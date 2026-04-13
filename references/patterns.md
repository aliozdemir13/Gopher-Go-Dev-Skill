# Go Patterns Reference

Detailed patterns for specific domains. The main SKILL.md covers when to read each section.

---

## Web APIs & HTTP

### HTTP handlers

Use the standard `net/http` patterns. Since Go 1.22, the default mux supports method
matching and path parameters:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", s.getUser)
mux.HandleFunc("POST /users", s.createUser)
```

### Handler structure

Keep handlers thin — they should parse input, call business logic, and write output:

```go
func (s *Server) getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    user, err := s.store.GetUser(r.Context(), id)
    if errors.Is(err, ErrNotFound) {
        http.Error(w, "user not found", http.StatusNotFound)
        return
    }
    if err != nil {
        http.Error(w, "internal error", http.StatusInternalServerError)
        s.logger.Error("getting user", "id", id, "err", err)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    if err := json.NewEncoder(w).Encode(user); err != nil {
        s.logger.Error("encoding response", "err", err)
    }
}
```

### Middleware

Middleware follows the `func(http.Handler) http.Handler` pattern:

```go
func logging(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            next.ServeHTTP(w, r)
            logger.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "duration", time.Since(start),
            )
        })
    }
}
```

### JSON encoding/decoding

- Use `json.NewEncoder`/`json.NewDecoder` for streaming.
- Limit request body size: `http.MaxBytesReader(w, r.Body, maxBytes)`.
- Validate decoded input before using it.
- Use struct tags for field naming: `` `json:"user_id,omitempty"` ``.

### Graceful shutdown

Always handle graceful shutdown in production servers:

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("server error: %v", err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    log.Fatalf("shutdown error: %v", err)
}
```

---

## CLI Tools

### Structure

Use `flag` for simple CLIs. For anything with subcommands, consider a library like
`cobra` or `urfave/cli`, but don't reach for them until you need them.

```go
func main() {
    var (
        addr    = flag.String("addr", ":8080", "listen address")
        verbose = flag.Bool("v", false, "verbose output")
    )
    flag.Parse()

    if err := run(*addr, *verbose); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}
```

### The `run` pattern

Keep `main` tiny. Put real logic in a `run` function that returns an error — this makes
the program testable and keeps exit handling in one place.

```go
func run(addr string, verbose bool) error {
    logger := newLogger(verbose)
    db, err := openDB()
    if err != nil {
        return fmt.Errorf("opening database: %w", err)
    }
    defer db.Close()

    srv := newServer(logger, db)
    return srv.ListenAndServe(addr)
}
```

### Output conventions

- Normal output goes to `os.Stdout`.
- Errors and diagnostics go to `os.Stderr`.
- Use exit code 0 for success, 1 for general errors, 2 for usage errors.

---

## Testing Deep-Dive

### Test helpers

Extract repeated setup into helpers. Mark them with `t.Helper()` so failure messages
point to the right line:

```go
func newTestServer(t *testing.T) *Server {
    t.Helper()
    s, err := NewServer(WithPort(0))
    if err != nil {
        t.Fatalf("creating test server: %v", err)
    }
    t.Cleanup(func() { s.Close() })
    return s
}
```

### Interfaces for test doubles

Define a minimal interface at the test call site (or use the one from production code),
then provide a simple fake:

```go
type fakeStore struct {
    users map[string]*User
}

func (f *fakeStore) GetUser(_ context.Context, id string) (*User, error) {
    u, ok := f.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}
```

Prefer fakes (simple in-memory implementations) over mocks with assertion libraries.
Fakes are easier to read, don't couple tests to call sequences, and work as documentation
of the interface contract. Use mocking libraries only when the interaction pattern itself
is what you're testing.

### Integration tests

Use build tags or `testing.Short()` to separate fast unit tests from slower integration tests:

```go
func TestDatabaseIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // ... test against real database
}
```

Use `t.Setenv` (Go 1.17+) for environment-dependent tests and `t.TempDir()` for
file-system tests — both clean up automatically.

### Benchmarks

Write benchmarks for performance-sensitive code:

```go
func BenchmarkEncodeJSON(b *testing.B) {
    data := makeLargePayload()
    b.ResetTimer()

    for b.Loop() {
        if err := json.NewEncoder(io.Discard).Encode(data); err != nil {
            b.Fatal(err)
        }
    }
}
```

Run with `go test -bench=. -benchmem` to include allocation stats. Use `b.ReportAllocs()`
if you only care about allocations for specific benchmarks.

### Test organization

- One `_test.go` file per source file, in the same package (for white-box tests) or in
  `package foo_test` (for black-box tests that verify the public API).
- Name test functions after the behavior: `TestHandler_ReturnsNotFoundForMissingUser` not
  `TestHandler1`.
- Use `testdata/` directory for fixture files — Go tooling ignores it automatically.

---

## Logging

Use `log/slog` (Go 1.21+) for structured logging:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.Info("server started", "addr", addr, "version", version)
logger.Error("query failed", "err", err, "query", q)
```

Pass the logger as a dependency — don't use the global `slog.Default()` in library code.
