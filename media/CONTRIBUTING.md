# Contributing to Maximo API Client

This repository is maintained as a closed-source project. The guidance below is for internal development and approved collaborators only.

## Getting Started

1. Clone the approved repository copy.
2. Install dependencies: `npm install`
3. Create a feature branch: `git checkout -b feature/your-feature-name`
4. Confirm you have approval to work on the requested change.

## Development Workflow

### Running Tests

```bash
# Run deterministic unit tests (default)
npm test

# Run integration tests explicitly (requires local test config/env)
npm run test:integration

# Run tests with coverage
npm run test:coverage

# Run tests in watch mode
npm run test:watch
```

### Code Quality

```bash
# Lint your code
npm run lint

# Auto-fix linting issues
npm run lint:fix

# Format your code
npm run format

# Check formatting
npm run format:check
```

### Building

```bash
# Compile TypeScript
npm run build

# Verify no secrets leaked in tracked test sources
npm run secrets:scan

# Verify npm package contents are publish-safe
npm run pack:check

# Watch mode for development
npm run watch
```

### Integration Test Secrets

- Never commit real credentials, API keys, or private certificates.
- Store local integration credentials in `src/tests/constants/test-configs.local.ts` (gitignored).
- Use `src/tests/constants/test-configs.local.example.ts` as a template.
- For ad-hoc execution, environment variables are also supported by `src/tests/constants/test-configs.ts`.

## Code Style

- **Language**: TypeScript with strict mode enabled
- **Formatting**: Prettier (120 char line width, single quotes, 2-space indent)
- **Linting**: ESLint with TypeScript rules
- **Naming Conventions**:
  - Classes: PascalCase
  - Functions/Methods: camelCase
  - Constants: UPPER_SNAKE_CASE
  - Interfaces: PascalCase with descriptive names

### Code Quality Guidelines

1. **Type Safety**: Avoid using `any` types when possible
2. **Documentation**: Add JSDoc comments for public APIs
3. **Testing**: Write tests for new features and bug fixes
4. **Error Handling**: Use proper error types and messages
5. **Backward Compatibility**: Don't break existing APIs

## Submitting Changes

1. Ensure all tests pass: `npm test`
2. Ensure code is formatted: `npm run format`
3. Ensure no linting errors: `npm run lint`
4. Commit your changes with a clear message:
   ```bash
   git commit -m "feat: add new feature X"
   # or
   git commit -m "fix: resolve issue with Y"
   ```
5. Push to the approved remote branch.
6. Open a Pull Request or internal review request with:
   - Clear description of changes
   - Reference to related issues (if any)
   - Test coverage for new code

## Commit Message Format

We follow conventional commits:

- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation changes
- `style:` Code style changes (formatting, etc.)
- `refactor:` Code refactoring
- `test:` Adding or updating tests
- `chore:` Maintenance tasks

## Questions?

Use the project’s approved internal communication or review channel for:
- Bug reports
- Feature requests
- Questions about usage
- Discussion about improvements

## License

By contributing, you assign or license your changes according to the project owner’s private licensing terms and any separate written agreement that governs this repository.
