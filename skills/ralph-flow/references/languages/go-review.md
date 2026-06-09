# Go Review Pack

Use this pack only for Go diffs. It complements role reviewers and focuses on Go-specific correctness, maintainability, and review hygiene.

## Always Check

- Error chains: wrap propagated errors with `%w` when callers may inspect them; preserve sentinel errors used with `errors.Is`/`errors.As`; avoid log-and-return duplicates.
- Context propagation: pass caller `ctx`; add bounded timeouts for external calls when no stronger deadline exists; call `cancel()` for derived contexts.
- HTTP clients: external `http.Client` instances need a timeout or an explicit, justified deadline path; requests should use `NewRequestWithContext`.
- Resource cleanup: close response bodies and rows; handle `rows.Err`; avoid `defer` in unbounded loops unless intended.
- String enums: repeated string modes, switch values, request source types, statuses, and actions should use typed constants when values cross function boundaries.
- Nil and zero values: guard nil maps, nil pointers, typed nil interfaces, nil channels, append aliasing, slice/map mutation side effects, and unsafe numeric conversions.
- API compatibility: do not silently change public JSON, storage formats, metrics, logs, errors, status codes, or protocols.
- External API contracts: when code accepts HTTP status codes or parses third-party responses, compare against existing local clients and official docs when needed.
- Tests: changed behavior should have focused tests or clear existing coverage; prefer behavior assertions over only mock-call assertions; use table tests where useful.
- Dependency hygiene: new dependencies need clear justification; keep `go.mod`, `go.sum`, and `vendor/modules.txt` consistent with repo workflow.

## Targeted Skill Routing

Read targeted Go skill instructions only when the diff touches that area. Do not load every Go skill by default.

- Any Go diff: `golang-code-style`, `golang-error-handling`, `golang-testing`, `golang-safety`.
- `context.Context`, deadlines, HTTP requests: `golang-context`.
- Goroutines, channels, locks, atomics, worker pools: `golang-concurrency`.
- SQL, transactions, `database/sql`, `sqlx`, `pgx`, migrations: `golang-database`.
- Auth, cookies, tokens, crypto, filesystem, process execution, user input: `golang-security`.
- Struct/interface design, struct tags, type assertions/switches: `golang-structs-interfaces`.
- Constants, enums, exported identifiers, constructors, receivers, errors: `golang-naming`.
- Go module or vendor changes: `golang-dependency-management`.
- Test files or testify: `golang-testing`, `golang-stretchr-testify`.
- Logs, metrics, tracing, alerts: `golang-observability`.
- CLI code, flags, Cobra, Viper: `golang-cli`, `golang-spf13-cobra`, `golang-spf13-viper`.
- gRPC/protobuf: `golang-grpc`.
- `fx`, `dig`, `wire`, `samber/do`: matching dependency-injection skill.
- Benchmarks, pprof, performance claims: `golang-benchmark`, `golang-performance`.

## Reporting

Report only concrete problems visible in the diff or adjacent code. Human-review hygiene can be `MINOR`, but still report it when it is likely to become a PR comment.

Examples worth reporting:

- `MINOR: file.go:42 — sentinel error is remapped without %w; callers cannot use errors.Is; wrap with fmt.Errorf("context: %w", err).`
- `MINOR: file.go:77 — external HTTP client has no timeout; request can hang until parent context cancellation; set a client timeout or derive a bounded context.`
- `MINOR: file.go:91 — switch uses raw string mode values across functions; typo compiles and fails only at runtime; introduce a typed string enum.`
- `MAJOR: file.go:120 — response body is not closed; leaks connections under load; defer resp.Body.Close() after nil-error response.`
