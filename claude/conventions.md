# Code Conventions

Extracted from actual source files. Imperative form — follow these patterns.

## Go (MessageForge.Backend, MessageForge.PoC)

### Error Handling
- Wrap errors with context: `fmt.Errorf("context: %w", err)`
- Never log AND return — choose one. Log only at top level (main.go).
- Return early on errors, keep happy path un-indented.

### Logging
- Use `log/slog` with `slog.NewJSONHandler`. Never `log` or `fmt.Println`.
- Fields: `slog.Error("msg", "error", err)` — structured, not string concat.

### Naming
- Packages: lowercase, concise (`database`, `server`, `telegram`)
- Files: underscores (`handlers_auth.go`, `pool_test.go`), plural nouns (`handlers.go`, `models.go`)
- Receivers: short, 1-2 chars (`func (s *Server)`, `func (a *App)`)
- Exported: PascalCase. Unexported: camelCase.
- FORBIDDEN: `interfaces.go` files

### Imports
- Group: stdlib first, blank line, external packages.

### Function Order
1. Types, interfaces, constants
2. Methods
3. Constructors (`New*`) — always at end of file

### Concurrency
- `oklog/run` for service lifecycle (execute/interrupt pairs)
- `errgroup` for bounded worker pools within a service
- Never `go func()` for service lifecycle

### Config
- `caarlos0/env` struct tags + `go-playground/validator`
- All secrets from env vars, never hardcoded

## TypeScript/React (MessageForge.PoC/web)

### Components
- PascalCase files matching component name (`AuthGuard.tsx`)
- State: `const [x, setX] = useState()` pattern
- Loading/error/empty states: always handle all three

### Imports
- React/framework first, then internal relative paths

### API Calls
- Use typed `apiFetch<T>()` wrapper, never raw `fetch()`
- Handle 429 (rate limit) with retry-after

### Linting
- ESLint flat config (v9+), TypeScript strict mode, React hooks plugin

## Apex (MessageForge.Salesforce)

### Naming
- Classes: PascalCase with suffix (`*Service`, `*Handler`, `*Validator`)
- Methods: camelCase
- Tests: `{ClassUnderTest}Test`, methods: `test{Scenario}`
- Constants: `UPPER_SNAKE_CASE`

### Security
- Constant-time HMAC comparison (XOR-based, not `==`)
- Protected Custom Metadata for secrets (`Middleware_Config__mdt`)
- `with sharing` on all classes by default
- Bind variables in SOQL, never string concatenation

### Testing
- `Test.startTest()` / `Test.stopTest()` for governor limit reset
- Bulk test with 200+ records
- `System.assertEquals(expected, actual, message)`
- Never `@SeeAllData=true`

### DML
- Collect in lists, operate outside loops (bulkify)
- Use `EventBus.TriggerContext.currentContext().setResumeCheckpoint()` for PE handlers

## Git Commits

Format: `<type>(<scope>): <description>`
- Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `security`
- Scopes: `poc`, `backend`, `salesforce`
- Lowercase, imperative mood, present tense
