# Go Interview Prep

A focused, sub-topic-per-file reference for Go developer interviews — from language basics to advanced runtime topics, with idiomatic patterns and the gotchas that come up in real interviews.

## How to use

1. **Skim the indexes** ([core-concepts.md](core-concepts.md), [concurrency.md](concurrency.md), [advanced-topics.md](advanced-topics.md), [web-development.md](web-development.md), [design-patterns.md](design-patterns.md)) to see what's covered.
2. **Drill into sub-files** for the topics you're weakest on. Each sub-file is ~150–300 lines with code, Q&A, and pitfalls.
3. **Day before:** review [revision.md](revision.md) — rapid-fire cheatsheet.
4. **Day of:** keep the [plan.md](plan.md) study tracker handy.

Recommended study order (4 weeks):

| Week | Focus |
|------|-------|
| 1 | Core concepts (syntax, structs, interfaces, errors, slices/maps, modules, stdlib) |
| 2 | Concurrency (goroutines, channels, select+context, sync, patterns) |
| 3 | Web (net/http, middleware, routing, DB, shutdown, gRPC) |
| 4 | Advanced (generics, reflection, unsafe, profiling, testing, GC) + design patterns |

## Index — Existing Files

| File | What it covers |
|------|---------------|
| [plan.md](plan.md) | 4-week study plan, topic checklist |
| [revision.md](revision.md) | Quick-fire cheatsheet — day-before review |
| [core-concepts.md](core-concepts.md) | Index → language fundamentals |
| [concurrency.md](concurrency.md) | Index → goroutines, channels, sync |
| [advanced-topics.md](advanced-topics.md) | Index → generics, reflection, profiling, GC |
| [web-development.md](web-development.md) | Index → HTTP, middleware, DB, gRPC |
| [design-patterns.md](design-patterns.md) | Index → idiomatic Go patterns |

## Core Concepts (split files)

| File | Topic |
|------|-------|
| [01-syntax-and-types.md](01-syntax-and-types.md) | Variables, constants, iota, types, pointers |
| [02-structs-and-interfaces.md](02-structs-and-interfaces.md) | Structs, methods, embedding, interfaces |
| [03-functions-and-closures.md](03-functions-and-closures.md) | Variadic, multiple returns, defer, closures |
| [04-error-handling.md](04-error-handling.md) | error interface, errors.Is/As, %w, panic/recover |
| [05-slices-arrays-maps.md](05-slices-arrays-maps.md) | Slice internals, aliasing, map quirks |
| [06-packages-and-modules.md](06-packages-and-modules.md) | go.mod, replace, vendor, internal |
| [07-stdlib-essentials.md](07-stdlib-essentials.md) | fmt, io, bufio, time, context, encoding/json |

## Concurrency (split files)

| File | Topic |
|------|-------|
| [goroutines.md](goroutines.md) | go keyword, GMP, stacks, leaks |
| [channels.md](channels.md) | Buffered vs unbuffered, close, range, nil |
| [select-and-context.md](select-and-context.md) | select, timeouts, context.Context |
| [sync-primitives.md](sync-primitives.md) | Mutex, RWMutex, WaitGroup, Once, Cond, Map, Pool |
| [concurrency-patterns.md](concurrency-patterns.md) | Worker pool, pipeline, fan-in/out, errgroup |

## Web Development (split files)

| File | Topic |
|------|-------|
| [net-http-basics.md](net-http-basics.md) | http.Handler, ServeMux 1.22+, http.Client |
| [middleware-pattern.md](middleware-pattern.md) | func(http.Handler) http.Handler, chaining |
| [routing-frameworks.md](routing-frameworks.md) | chi, gin, echo, gorilla/mux, fiber |
| [db-access.md](db-access.md) | database/sql, sqlx, pgx, sqlc, GORM |
| [graceful-shutdown.md](graceful-shutdown.md) | signal.NotifyContext, srv.Shutdown |
| [grpc-in-go.md](grpc-in-go.md) | Protobuf, streaming, interceptors |

## Advanced Topics (split files)

| File | Topic |
|------|-------|
| [generics.md](generics.md) | Type parameters, constraints, ~T, comparable |
| [reflection.md](reflection.md) | reflect.Type/Value, tags, performance |
| [unsafe-and-cgo.md](unsafe-and-cgo.md) | unsafe.Pointer, cgo, CGO_ENABLED |
| [profiling-and-tracing.md](profiling-and-tracing.md) | pprof, go tool trace, runtime metrics |
| [testing-and-benchmarks.md](testing-and-benchmarks.md) | testing.T, table-driven, fuzzing, benchstat |
| [gc-and-memory.md](gc-and-memory.md) | Tri-color GC, escape analysis, GOMEMLIMIT |

## Design Patterns (split files)

| File | Topic |
|------|-------|
| [functional-options-pattern.md](functional-options-pattern.md) | THE Go idiom for configurable constructors |
| [factory-pattern-go.md](factory-pattern-go.md) | Interface + switch / driver registration |
| [singleton-pattern-go.md](singleton-pattern-go.md) | sync.Once / sync.OnceValue |
| [decorator-pattern-go.md](decorator-pattern-go.md) | Wrap impls for logging/caching/retries |
| [strategy-pattern-go.md](strategy-pattern-go.md) | Inject interface for swappable algorithms |
| [observer-pattern-go.md](observer-pattern-go.md) | Channels + event bus |
| [repository-pattern-go.md](repository-pattern-go.md) | Hide data access behind interfaces |

## Related study material

- [../nodejs/](../nodejs/) — Node.js / JavaScript prep (async, event loop comparison)
- [../spring-detailed/](../spring-detailed/) — Java / Spring deep-dive
- [../design-patterns/](../design-patterns/) — Language-agnostic patterns
- [../design-patterns/golang/](../design-patterns/golang/) — More Go-specific pattern examples
- [../serious-prep/](../serious-prep/) — System design + behavioral interview material

## Key Go Proverbs (memorize)

- "Don't communicate by sharing memory; share memory by communicating."
- "The bigger the interface, the weaker the abstraction."
- "Make the zero value useful."
- "Errors are values."
- "Accept interfaces, return structs."
- "Clear is better than clever."
- "A little copying is better than a little dependency."

## External references

- [Go Playground](https://go.dev/play/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Go by Example](https://gobyexample.com/)
- [Standard library docs](https://pkg.go.dev/std)
- [Awesome Go](https://github.com/avelino/awesome-go)
