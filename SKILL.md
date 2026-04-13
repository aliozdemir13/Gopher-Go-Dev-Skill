---
name: gopher
description: >
  Idiomatic Go development guidance and code generation. Use this skill whenever the user asks
  you to write, review, refactor, debug, or improve Go code — including packages, services,
  CLI tools, web APIs, microservices, or any .go file. Also trigger when the user asks about
  Go best practices, error handling patterns, concurrency design, interface design, testing
  strategies, or performance optimization in Go. Trigger on mentions of "Go", "Golang",
  "goroutine", "channel", ".go file", "go mod", "go test", "go build", or any Go standard
  library package. Even if the user just pastes Go code without explicit instructions, use
  this skill to inform your review and suggestions.
---

# Idiomatic Go Development

This skill makes Claude a Go expert that writes code the way experienced Go developers do —
simple, readable, and aligned with the conventions of the Go ecosystem. The philosophy is
"clear is better than clever."

**For web APIs, CLI tools, testing deep-dives, and logging patterns**, read
`references/patterns.md` alongside this file when working in those domains.

## Core Principles

These reflect the values the Go community has converged on over 15+ years. When in doubt,
lean toward the simpler option.

1. **Simplicity over abstraction.** Don't introduce a pattern (factory, builder, DI
   framework) unless the code is genuinely hard to maintain without it. A few lines of
   repeated setup code is often better than a clever generic helper.

2. **Explicit over implicit.** Return errors rather than panicking. Pass dependencies as
   arguments rather than relying on package-level globals. Make zero values useful rather
   than requiring constructors.

3. **Small interfaces.** Define interfaces where they're consumed, not where they're
   implemented. One or two methods is ideal. `io.Reader` and `io.Writer` are the gold
   standard.

4. **Composition over inheritance.** Embed structs and interfaces to compose behavior.
   Go doesn't have inheritance, and idiomatic Go doesn't try to simulate it.

5. **Package design matters.** A package should provide a focused capability with a clear
   API. Avoid "utils", "helpers", or "common" packages. Name packages after what they
   *provide*, not what they *contain*. 
   Use the internal/ directory to enforce encapsulation. 
   Code inside internal/ cannot be imported by other modules, which allows you to change private implementation details without breaking the public API.

---

## Naming

Go's naming conventions matter because exported names are part of the API and the package
name qualifies every use.

**Packages:** Short, lowercase, single-word when possible. The package name is part of every
call site — `http.Client` reads better than `httputil.HTTPClient`. Never use `util`,
`common`, `base`, `shared`, or `misc`.

**Variables and functions:** `camelCase`. Short names for short scopes (`i`, `r`, `ctx`).
Longer, descriptive names for wider scopes. Receivers are one or two letters from the type
name (`s` for `Server`). Don't use `self` or `this`.

**Interfaces:** Name after the behavior, typically ending in `-er`: `Reader`, `Writer`,
`Handler`. For multi-method interfaces, use a noun describing the capability: `Store`.

**Exported names:** Avoid stuttering. `http.Server` not `http.HTTPServer`. If you repeat
the package name in the type name, something is wrong.

**Acronyms:** All-caps: `ID`, `HTTP`, `URL`, `API`, `JSON`, `SQL`. When combined:
`userID`, `httpClient`.

**Errors:** Variables use `Err` prefix: `ErrNotFound`. Types use `Error` suffix:
`PathError`.

---

## Error Handling

Every error either gets handled meaningfully or gets wrapped with context and returned.

```go
// Good — wrap with context using %w
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doing something for user %s: %w", userID, err)
}

// Bad — bare return loses context
if err != nil {
    return err
}

// Bad — %v breaks the error chain (use %w)
if err != nil {
    return fmt.Errorf("failed: %v", err)
}
```

**Guidelines:**
- **Wrap with `%w`** so callers can use `errors.Is` / `errors.As`. Use `%v` only to
  intentionally break the chain.
- **Add debugging context.** Include the operation and identifiers:
  `"querying user %s: %w"`. Think about what you'd want in a log at 3am.
- **Don't say "failed to".** The caller knows it failed — say what was attempted.
- **Sentinel errors** for conditions callers check: `var ErrNotFound = errors.New("not found")`
- **Error types** when callers need structured info:
  ```go
  type ValidationError struct {
      Field, Message string
  }
  func (e *ValidationError) Error() string {
      return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
  }
  ```
- **Never panic in library code.** Reserve `panic` for unrecoverable programmer errors.
- **Handle at the right level.** Low-level code wraps and returns. High-level code (HTTP
  handlers, CLI commands) decides what to do.

---

## Concurrency

**Don't start a goroutine unless you know how it will stop.**

### Goroutine lifecycle

Every goroutine should have a clear termination condition. The parent should be able to
wait for it or signal it to stop via `context.Context`.

```go
func processItems(ctx context.Context, items <-chan Item) error {
    g, ctx := errgroup.WithContext(ctx)
    for item := range items {
        g.Go(func() error {
            return process(ctx, item)
        })
    }
    return g.Wait()
}
```

As of Go 1.22, loop variables are created per-iteration. You no longer need the v := v trick when starting goroutines inside a for loop.

### Channels vs mutexes

- **Channels** for ownership transfer — sending a value transfers responsibility for it.
- **Mutexes** for shared state — if goroutines read/write the same data without needing
  communication semantics, `sync.Mutex` is clearer.
- **Always `select` with `ctx.Done()`** for anything that blocks.

### Common patterns

- `errgroup.Group` — structured concurrency, first error cancels all.
- `sync.WaitGroup` — just wait for goroutines to finish.
- `sync.Once` — lazy initialization.
- Worker pools with a semaphore channel for bounding concurrency.

### Context

`context.Context` should be the first parameter of any function that does I/O, might block,
or needs cancellation. Always pass it through — don't store it in a struct.

---

## Interface Design

### Accept interfaces, return structs

```go
// Good — accepts any reader, returns concrete type
func ParseConfig(r io.Reader) (*Config, error) { ... }

// Bad — returns interface, hiding the concrete type for no reason
func NewService() ServiceInterface { ... }
```

### Define interfaces at the consumer

Unlike Java, define interfaces where they're *used*, not where they're *implemented*:

```go
// In package "handler" (consumer), not package "store" (provider)
type UserStore interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
```

### Keep interfaces small

One to three methods is the sweet spot. More than five usually means you're describing a
concrete type, not a behavior.

### When NOT to use an interface

- Only one implementation, no testing concern.
- Wrapping a concrete type just to have an interface.
- Configuration or option structs.

---

## Struct Design

### Zero value usefulness

Design structs so the zero value is valid. `sync.Mutex`, `bytes.Buffer`, and `http.Client`
all work at their zero value — your types should too where possible.

```go
type Cache struct {
    mu      sync.Mutex
    entries map[string]entry
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.entries == nil {
        return "", false
    }
    // ...
}
```

### Functional options

For structs with many optional configurations:

```go
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

Use when there are 3+ optional settings that may grow. For simpler cases, use direct
parameters or a small config struct.

### Nil over Empty

Prefer var s []string (nil) over s := []string{} (empty) for initial declarations. A nil slice is a valid zero value that works with append, len, and cap without an immediate allocation.

---

## Testing

Good tests are table-driven, focused, and test behavior rather than implementation.

### Table-driven tests

The default pattern for multiple input/output scenarios:

```go
func TestParseSize(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {name: "bytes", input: "100B", want: 100},
        {name: "kilobytes", input: "2KB", want: 2048},
        {name: "empty string", input: "", wantErr: true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseSize(tt.input)
            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error, got nil")
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got != tt.want {
                t.Errorf("ParseSize(%q) = %d, want %d", tt.input, got, tt.want)
            }
        })
    }
}
```

For test helpers, fakes vs mocks, integration tests, benchmarks, and test organization,
see `references/patterns.md` → **Testing Deep-Dive**.

---

## Generics (Go 1.18+)

Use generics when they eliminate real code duplication across types. Don't use them just
because they exist.

**Good uses:** Data structures, utility functions across numeric/comparable types.
**Bad uses:** When only one or two types will ever be used, when interfaces already solve it.

```go
func Filter[S ~[]E, E any](s S, keep func(E) bool) S {
    var result S
    for _, v := range s {
        if keep(v) {
            result = append(result, v)
        }
    }
    return result
}
```

---

## Performance

Profile before optimizing. Rules of thumb for when it matters:

- Preallocate slices when you know the length: `make([]T, 0, n)`.
- `strings.Builder` for building strings in loops.
- Reuse buffers, use `sync.Pool` for short-lived objects under high load.
- Value receivers for small structs (<64 bytes) that don't need mutation.
- Profile with `pprof` before making performance claims.

---

## Anti-Patterns

Flag these when reviewing or refactoring:

| Anti-pattern | Better approach |
|---|---|
| `any` everywhere | Generics or concrete types |
| Stuttering names (`http.HTTPServer`) | `http.Server` |
| Giant interfaces (10+ methods) | Split into focused interfaces |
| `init()` with side effects | Explicit init in `main` |
| Package-level mutable globals | Pass as dependency |
| Returning interface from constructor | Return concrete type |
| `select {}` without `ctx.Done()` | Always include cancellation |
| `time.Sleep` in production | `time.After` with `select` or `time.Ticker` |

---

## Code Review Checklist

1. Are errors wrapped with `%w` and checked with `errors.Is`/`errors.As`?
2. Does every goroutine have a clear shutdown path?
3. Are interfaces small and defined at the consumer?
4. Is `context.Context` threaded through I/O paths?
5. Are names clear and non-stuttering?
6. Are tests table-driven where appropriate?
7. Is the zero value of exported structs useful (or documented)?
8. Run `go test -race` — any data races?
9. Is `defer` used for cleanup?
10. Could any complex code be simplified?
