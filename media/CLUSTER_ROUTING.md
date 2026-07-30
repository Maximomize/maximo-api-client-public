# Cluster routing & JVM pinning

In a Maximo cluster (typically MAS / OpenShift), an API call lands on whichever
JVM the load balancer picks. That's fine for stateless reads, but problematic
when you actually need a *specific* JVM — for example:

- Pulling logs or stack traces from the same JVM a user is logged into.
- Triggering a cache flush on every node in the cluster.
- Reproducing a JVM-local bug that only manifests on one pod.

The `cluster-routing` feature adds:

- **`ClusterService`** — discover cluster topology and locate user sessions across JVMs.
- **`MaximoClient.pinTo(jvm)`** — get a child client whose every call lands on a chosen JVM.
- **`MAXIMOMIZE.CLUSTER`** (optional Jython REST scripthandler) — pod-internal
  HTTP relay that powers the pinning when no per-pod ingress is available.

Below is the conceptual model, the install path, and a usage tour with copy-paste
examples.

---

## The routing constraint (read this first)

A load balancer in front of Maximo (IHS+WAS plugin / NGINX / OCP route /
Kubernetes Service) picks a JVM on first request and uses `JSESSIONID`
clone-suffix affinity for follow-ups. You **cannot**, from an external client,
force a request onto a chosen JVM by manipulating cookies — Manage doesn't
replicate `HttpSession` across JVMs, so a rewritten clone suffix routes the
plugin to JVM X but the request 401s.

Three real mechanisms exist, in order of preference:

| Mechanism | Needs autoscript? | Needs per-pod ingress? | Works for |
|---|---|---|---|
| **MMI cross-forward** | No | No | `/mmi/members/{id}/*` only — the MMI servlet already cross-forwards internally |
| **Per-pod ingress URL** | No | Yes | Arbitrary OSLC calls — clean but deployment-dependent |
| **Autoscript relay** | Yes (`MAXIMOMIZE.CLUSTER`) | No | Arbitrary OSLC calls — works anywhere the LB does |

`pinTo()` picks the best mechanism that's available and falls back gracefully.
For monitoring across the cluster (threads, memory, GC, logger config) you
don't need pinning at all — the per-member MMI endpoints already do this.

---

## Quick tour

```ts
import { MaximoClient, AuthType } from '@maximomize/maximo-api-client';

const client = new MaximoClient({
    baseUrl: 'maximo.example.com',
    contextRoot: 'maximo',
    apiHome: 'api',
    authType: AuthType.APIKEY,
    apiKey: '...',
});
await client.authenticate();

// 1. Discovery — works without any server-side install.
const here  = await client.cluster.here();              // which JVM answered this call?
const nodes = await client.cluster.listNodes();         // every JVM in the cluster
const jvm   = await client.cluster.findUserJvm('alice'); // which JVM holds Alice's session?

// 2. Pinning — requires the MAXIMOMIZE.CLUSTER autoscript installed,
//    OR a node entry that carries a `directUrl`.
if (jvm) {
    const pinned = await client.pinTo(jvm);

    // Every call on `pinned` lands on alice's JVM.
    const there = await pinned.cluster.here();          // verify
    const wo    = await pinned.workOrder.fetch({ wonum: '1001' });
    const mem   = await pinned.mmi.getMemory();         // memory of the JVM alice is on
}
```

---

## Discovery — `ClusterService`

### `client.cluster.here(): Promise<MaximoHere & { memberId? }>`

Returns the identity of the JVM that answered the current call. Uses
`/mmi/members/thisserver` under the hood. Useful for verifying that a pinned
client actually landed on the requested node.

```ts
const here = await client.cluster.here();
// { hostname: '10.0.0.3', ip: null, nodeName: 'MAXIMO', memberId: 'MAXIMO' }
```

### `client.cluster.listNodes(opts?): Promise<MaximoJvm[]>`

Enumerates every JVM the cluster knows about, from `/mmi/members`. Tolerates
the differing payload shapes across Maximo versions (MAS `member[]`,
older `members[]`, very-old `rdfs:member[]`).

```ts
const nodes = await client.cluster.listNodes();
// [
//   { serverHost: '10.0.0.1:13400', nodeName: 'MAXIMO',
//     jvmTag: 'aim-dev-main-default-7c7648db88-p8vbd',
//     ip: null, memberId: 'MTAuMTI4LjE4...', isHere: false },
//   ...
// ]
```

Pass `{ userId }` to mark which nodes hold an active `MAXSESSION` row for that
user. The user lookup tries OSLC `MXAPIMAXSESSION` first and falls back to
the `MAXIMOMIZE.CLUSTER` autoscript if the object structure isn't exposed.

```ts
const nodes = await client.cluster.listNodes({ userId: 'alice' });
// each node now has `userActive: boolean`
```

**`isHere` resolution rules** (the gotcha): if the server-side payload
explicitly flags an entry with `thisServer: true`, that wins. Otherwise we
fall back to a host comparison against `/mmi/members/thisserver` — and we
deliberately ignore an explicit `thisServer: false`, because in MAS the
`/members` call is often answered by a bundle JVM that lists every worker
with `thisServer: false` even though one of those workers really did answer
the separate `/thisserver` call.

### `client.cluster.findUserJvm(userId): Promise<MaximoJvm | null>`

Returns the first JVM that currently holds an `HttpSession` for the user, or
`null` if no JVM does.

### `client.cluster.findUserJvms(userId): Promise<MaximoJvm[]>`

Multi-session variant — a user may be logged into more than one JVM
(multiple browser tabs, mobile + desktop, etc.).

### `client.cluster.resolveNode(identifier): Promise<MaximoJvm>`

Resolve a string into a `MaximoJvm`. Accepts a node name, host, IP, JVM tag,
or member id — case-insensitive, port-tolerant. Throws
`ClusterError.UnknownNode` if nothing matches.

```ts
await client.cluster.resolveNode('aim-dev-main-default-7c7648db88-p8vbd');
await client.cluster.resolveNode('10.0.0.1');         // bare IP fine
await client.cluster.resolveNode('10.0.0.1:13400');   // port-suffixed fine
```

### `client.cluster.getScriptInfo(): Promise<ClusterScriptInfo | null>`

Probes the `MAXIMOMIZE.CLUSTER` autoscript via `?action=ping`. Cheap — no
Mbo or DNS work. Returns version + protocol info if installed, `null` if
not. Memoised for the lifetime of the service instance.

```ts
const info = await client.cluster.getScriptInfo();
// { version: '1.0.0', protocol: 1, node: {...}, versionKnown: true } | null
```

Forward-compat: if `?action=ping` returns 400 (older script that doesn't know
the action), we fall back to `?action=nodes` to confirm presence and return a
`{ versionKnown: false }` placeholder — so a newer client still works against
an older script.

### `client.cluster.isScriptAvailable(): Promise<boolean>`

Sugar over `getScriptInfo() !== null`. Same memoisation.

---

## Pinning — `MaximoClient.pinTo()`

```ts
async pinTo(jvm: MaximoJvm | string): Promise<MaximoClient>
```

Returns a *new* `MaximoClient` whose every call lands on the chosen JVM.
The selection of mechanism is automatic:

1. **`direct-url`** — if the resolved `MaximoJvm` has `directUrl` set
   (per-pod ingress), the child client targets that URL and performs a
   fresh `authenticate()` against it.
2. **`autoscript-relay`** — otherwise, calls are routed through
   `MAXIMOMIZE.CLUSTER` which forwards pod-internally to the target on
   port `:9080`. Inbound method/body/auth-headers are passed through
   unchanged.

If neither is available, throws `ClusterError.CannotPin`. If the current
client uses session-cookie auth (`LDAP_FORM` / `MAS_USER_PASS`), throws
`ClusterError.CookieAuthNotPinnable` — the cookie is bound to the ingress
JVM and can't be replayed against a different one. Use `apikey` for pinned
operations.

```ts
const target = await client.cluster.findUserJvm('alice');
if (!target) throw new Error('alice not logged in');

const pinned = await client.pinTo(target);

// All of these now run against alice's JVM:
const there  = await pinned.cluster.here();
const wo     = await pinned.workOrder.fetch({ wonum: '1001' });
const threads = await pinned.mmi.getThreadDump();
```

### Verifying the pin

Every relayed response carries three diagnostic headers:

| Header | Value |
|---|---|
| `X-Maximomize-Node` | The pod that ran the script (LB choice). |
| `X-Maximomize-Relay-Target` | The pod whose response body you're seeing. Presence = response came through the relay. |
| `X-Maximomize-Relay-Endpoint` | The upstream path that was forwarded. |

You can read these from any axios response object:

```ts
const resp = await (pinned as any).getHttpClient()
    .get(`/maximo/api/members/thisserver`, { lean: '1' });

console.log(resp.headers['x-maximomize-relay-target']);
// "10.0.0.1"   ← the pinned JVM
```

If `X-Maximomize-Node === X-Maximomize-Relay-Target`, the LB happened to
route the relay request to the very pod you asked to pin to — the script
loopback'd to `127.0.0.1`, still a correct pin.

---

## What MMI calls *don't* need pinning

`/mmi/members/{id}/<metric>` (threads, memory, GC, pingdb, sysprops, loggers,
…) already cross-forwards JVM-to-JVM via the MMI servlet using RMI. So:

```ts
// No pinning needed. The MMI servlet on whichever JVM answers
// pulls the data from MXServer2 internally and returns it.
const threads = await client.mmi.getThreads('MXServer2');
const memory  = await client.mmi.getMemory('MXServer2');
```

The `pinTo` relay deliberately skips these paths so calls don't make a
wasteful double hop. Pinning is only for things the MMI API doesn't already
solve — arbitrary OSLC reads, action invocations, log tailing.

---

## The `MAXIMOMIZE.CLUSTER` autoscript

Ships in `resources/autoscripts/MAXIMOMIZE.CLUSTER.py`. A single Jython REST
scripthandler with three action-dispatched operations:

| Action | Method | Purpose |
|---|---|---|
| `nodes` | GET | List cluster nodes (full enumeration from `SERVERSESSION` + `MAXSESSION` join). |
| `whereis` | GET | Find which node(s) hold a user's `HttpSession`, when `MXAPIMAXSESSION` isn't exposed. |
| `ping` | GET | Lightweight availability + version probe. No Mbo or DNS work. Used by `getScriptInfo`. |
| `relay` | ANY | Pod-internal HTTP forward to `http://<target>:9080<endpoint>`. Inbound method/body/headers passed through unchanged. |

### Why a script at all?

A pure-REST client can do discovery (`/mmi/members`, `MXAPIMAXSESSION`) and
single-JVM monitoring (`/mmi/members/{id}/*`) without any server-side help.
The script earns its keep for **arbitrary OSLC pinning** — there is no
client-side trick that makes a workorder fetch land on a chosen JVM, so the
script does the relay server-side.

### Auth model — pass-through, no token minting

The relay reads the caller's existing auth header off the inbound request
(`apikey`, `maxauth`, or `Authorization`) and copies it onto the outbound
hop. The target JVM authenticates the same user the same way. **No
short-lived `APIKEYTOKEN` is created**, which keeps the script simple and
removes a class of cleanup-after-failure failure modes.

Caveat: if the client is using *session-cookie auth* (LDAP form / OIDC), the
cookie is bound to JVM A's `HttpSession` and won't authenticate on JVM B.
`MaximoClient.pinTo()` rejects this up front with
`ClusterError.CookieAuthNotPinnable` — switch to apikey for pinned operations.

### Install

You can install the autoscript programmatically via the client library:

```ts
// Idempotently installs or updates the bundled MAXIMOMIZE.CLUSTER autoscript
const result = await client.cluster.installScript({ verify: true });
console.log(result); // { ok: true, created: true, updated: false, verified: true, scriptName: 'MAXIMOMIZE.CLUSTER' }
```

Alternatively, run the included CLI installer script:

```bash
# .env.proxy / .env.cluster / .env at repo root with maximo_host + apikey
npx ts-node scripts/install-cluster-script.ts
```

Both methods idempotently deploy the script source bundled in `resources/autoscripts/MAXIMOMIZE.CLUSTER.py` using OSLC bulk `AddChange` requests.

### Jython constraints honoured

- No `import json` — JSON is built with `com.ibm.json.java.JSONObject` /
  `JSONArray`.
- HTTP via `java.net.URL` / `HttpURLConnection`.
- Host identity via `java.net.InetAddress` + `MXServer.getInstanceName()`.
- Mbo access via `psdi.server.MXServer` + `psdi.mbo.SqlFormat`.
- gzip / deflate response decoding via `java.util.zip`.

### Versioning

The ping payload exposes two version fields:

- `SCRIPT_VERSION` (currently `"1.1.0"`) — bumped on behaviour changes.
- `SCRIPT_PROTOCOL` (`1`) — bumped only on backward-incompatible wire-shape
  changes to relay/whereis payloads.

Version history:
- **1.0.0** — initial: `nodes` / `whereis` / `relay` (buffered) / `ping`.
- **1.1.0** — relay learned SSE / NDJSON pass-through (no body buffering for
  streaming content types); inbound PATCH is translated to POST +
  `x-method-override: PATCH` so it survives Java's `HttpURLConnection.setRequestMethod`
  (which rejects PATCH).

Clients compare against the value they were built for and can warn an
operator to redeploy when the script is too old. The client itself also
falls back gracefully if `?action=ping` is missing entirely.

### Streaming responses through the relay (SSE / NDJSON)

The relay detects streaming `Content-Type`s and switches from
"buffer everything via `_drain`" to "pipe upstream bytes straight to the
servlet output stream." The detection covers:

- `text/event-stream` (SSE)
- `application/x-ndjson`
- `application/stream+json`

For these, the relay writes directly to `request.getHttpServletResponse().getOutputStream()`
without setting the `responseBody` global — letting Maximo's scripthandler
finalize the response without overwriting the streamed bytes. The same
`X-Maximomize-Node` / `X-Maximomize-Relay-Target` / `X-Maximomize-Relay-Endpoint`
diagnostic headers are stamped before `flushBuffer()` so they reach the
client (headers must be set before the response is committed).

This means a pinned client can call any SSE-producing autoscript and the
stream comes back transparently. For example, with a Maximomize log-tail
script (`MAXIMOMIZE.LOGSTREAM`) installed:

```ts
const target = await client.cluster.findUserJvm('alice');
const pinned = await client.pinTo(target!);

// pinned client → cluster relay → target pod → MAXIMOMIZE.LOGSTREAM → SSE stream back
const resp = await (pinned as any).getHttpClient().axiosInstance.get(
    '/maximo/api/script/MAXIMOMIZE.LOGSTREAM',
    { params: { action: 'stream', timeout: '30' }, responseType: 'stream' }
);
// resp.data is a Node Readable; parse SSE frames as they arrive.
```

Two approaches to log streaming worth comparing:

1. **Via the cluster relay** (this code path). One install of `MAXIMOMIZE.CLUSTER`
   makes every SSE-producing script automatically pinnable. The relay
   adds a small fixed overhead per stream (intra-pod HTTP hop) but no
   per-frame cost — the bytes are forwarded as they arrive.
2. **Direct stream** (no relay). A client that targets the SSE script
   directly without going through the relay. Lower latency, no extra hop,
   but the client either accepts whichever JVM the LB picks or needs
   another mechanism (per-pod ingress, JSESSIONID clone affinity) to
   land on the chosen JVM.

Both have their place. The relay is the right answer when you want
"pin once, then call anything"; the direct stream wins when you only do
log tailing and want minimum latency.

### Notes about ports — RMI vs HTTP

If you've worked with Maximo 7.x you may remember `:13400` as the RMI
registry port. That's still in play — `SERVERSESSION.SERVERHOST` carries
the registry port because that's how pods find each other for *Java*
inter-JVM calls (cron coordination, MMI cross-forwarding, etc.).

Our relay does **HTTP**, not RMI, so the script strips the `:13400` and
appends `:9080` (the Liberty HTTP port) before forwarding. Both the script
and `MaximoClient.pinTo()` strip the port consistently — if you ever pass
a node identifier with `:13400` to `resolveNode()` it'll still match.

External (out-of-cluster) callers cannot reach `:13400` — the K8s Service
exposes only HTTP. That's why the *thick-client RMI patterns* from old
Maximo 7.x tutorials don't translate to MAS 9. The relay autoscript is the
MAS-shaped replacement for "I want this call to land on a specific JVM."

---

## Errors

`ClusterError` is thrown with a structured `code` field for branchable
handling:

```ts
import { ClusterError, ClusterErrorCode } from '@maximomize/maximo-api-client';

try {
    const pinned = await client.pinTo('not-a-real-node');
} catch (err) {
    if (err instanceof ClusterError) {
        switch (err.code) {
            case ClusterErrorCode.UnknownNode:        // bad identifier
            case ClusterErrorCode.CannotPin:          // no directUrl + no script
            case ClusterErrorCode.CookieAuthNotPinnable:
            case ClusterErrorCode.DiscoveryUnavailable:
            case ClusterErrorCode.RelayFailed:
                // handle...
        }
    }
}
```

---

## Tests

Unit tests (no live server required):

```bash
npx jest src/tests/cluster-service.test.ts
```

Integration tests against a real cluster (env-driven, skipped without
credentials):

```bash
# Required vars in .env.cluster / .env.proxy / .env at repo root:
#   maximo_host  = https://host.example.com
#   apikey       = <api key>
#   context_root = maximo   (optional, default)
#   apihome      = api      (optional, default)
npx jest --config jest.integration.config.ts \
         src/tests/integration/cluster-service.integration.test.ts
```

The integration suite is `describe.skip` when no env file is present, so
the file is safe to commit and safe to run in CI without secrets.
