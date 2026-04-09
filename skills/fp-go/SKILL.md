---
name: fp-go
description: >
  Active FP guidance for Go projects using fp-go/v2. Provides monad selection,
  code review, migration assistance, testing guidance, and pattern suggestions.
  Use when working on Go projects that use fp-go/v2 or fptest-go, migrating Go
  code to functional style, or asking about functional programming patterns in Go.
  Triggers on: "/fp-go", imports containing "IBM/fp-go" or "franchb/fptest",
  mentions of "fp-go", "fptest", "functional Go", "monad", "Either", "Option",
  "Result", "IOResult", "ReaderIOResult", "Effect", "property-based testing",
  "law verification", "algebraic laws" in Go context.
---

# fp-go/v2 — Active FP Guidance

fp-go/v2 (`github.com/IBM/fp-go/v2`) is a typed functional programming library for Go 1.24+. It provides Option, Either/Result, IO, and Effect monads with data-last, curried APIs designed for pipeline composition via `Pipe` and `Flow`. Two module families: **standard** (struct-based, full FP toolkit) and **idiomatic** (Go-native `(value, error)` tuples, zero-alloc, 2-32x faster).

---

## Import Conventions

| Alias | Package |
|-------|---------|
| `F`   | `github.com/IBM/fp-go/v2/function` |
| `O`   | `github.com/IBM/fp-go/v2/option` |
| `E`   | `github.com/IBM/fp-go/v2/either` |
| `R`   | `github.com/IBM/fp-go/v2/result` |
| `A`   | `github.com/IBM/fp-go/v2/array` |
| `IO`  | `github.com/IBM/fp-go/v2/io` |
| `IOR` | `github.com/IBM/fp-go/v2/ioresult` |
| `IOE` | `github.com/IBM/fp-go/v2/ioeither` |
| `RIO` | `github.com/IBM/fp-go/v2/context/readerioresult` |
| `EFF` | `github.com/IBM/fp-go/v2/effect` |
| `P`   | `github.com/IBM/fp-go/v2/pair` |
| `T`   | `github.com/IBM/fp-go/v2/tuple` |
| `N`   | `github.com/IBM/fp-go/v2/number` |
| `S`   | `github.com/IBM/fp-go/v2/string` |
| `B`   | `github.com/IBM/fp-go/v2/boolean` |
| `L`   | `github.com/IBM/fp-go/v2/optics/lens` |
| `PR`  | `github.com/IBM/fp-go/v2/optics/prism` |

**fptest-go** (property-based testing):

| Alias | Package |
|-------|---------|
| `FPT`  | `github.com/franchb/fptest/gen` |
| `FPTL` | `github.com/franchb/fptest/laws` |
| `FPTA` | `github.com/franchb/fptest/assert` |
| `FPTP` | `github.com/franchb/fptest/prop` |
| `FPTM` | `github.com/franchb/fptest/mock` |

**Idiomatic variants** (tuple-based, zero-alloc):

| Alias | Package |
|-------|---------|
| `IR`  | `github.com/IBM/fp-go/v2/idiomatic/result` |
| `IO_` | `github.com/IBM/fp-go/v2/idiomatic/option` |
| `IIR` | `github.com/IBM/fp-go/v2/idiomatic/ioresult` |
| `IRR` | `github.com/IBM/fp-go/v2/idiomatic/context/readerresult` |

## Monad Selection Decision Tree

- **Pure value** → use the value directly
- **May be absent** → `Option[A]`
- **Can fail with `error`** → `Result[A]` (= `Either[error, A]`)
  - Custom error type E → `Either[E, A]`
- **Lazy + can fail** → `IOResult[A]` = `func() Either[error, A]`
- **Needs `context.Context` + lazy + can fail** → `ReaderIOResult[A]` via `context/readerioresult`
- **Typed DI + context + lazy + can fail** → `Effect[C, A]` via `effect`
- **Performance-critical** → prefer `idiomatic/` variants

## Standard vs Idiomatic

| Aspect | Standard | Idiomatic |
|--------|----------|-----------|
| Representation | `Either[error, A]` struct | `(A, error)` tuple |
| Performance | Baseline | 2-32x faster, zero allocs |
| Custom error types | `Either[E, A]` for any E | `error` only |
| Go interop | Requires `Unwrap`/`Eitherize` | Native `(val, err)` |

**Rule**: Idiomatic for production hot paths. Standard for custom error types or advanced FP features.

## Key Rules

1. **Data-last**: Config/behavior params first, data last. Enables partial application.
2. **Type param ordering**: Non-inferrable first. `Ap[B, E, A]` — B leads because it can't be inferred.
3. **Flow = left-to-right** (use for pipelines). **Compose = right-to-left** (avoid in pipelines).
4. **Pipe vs Flow**: `F.Pipe3(value, f1, f2, f3)` applies immediately. `F.Flow3(f1, f2, f3)` creates a reusable function.
5. **Arity-numbered**: `Pipe1`-`Pipe20`, `Flow1`-`Flow20`. Match the count.
6. **Naming**: `Chain`=flatMap, `Map`=fmap, `Ap`=apply, `ChainFirst`/`Tap`=side effect, `ChainEitherK`=lift pure Either, `Of`=pure, `Fold`=catamorphism.
7. **Prefer `result` over `either`** unless custom error type needed.
8. **`result.Eitherize1(fn)`** wraps `func(X) (Y, error)` → `func(X) Result[Y]`. Up to `Eitherize15`.
9. **`function.Void`/`function.VOID`** instead of `struct{}`/`struct{}{}`.
10. **Go 1.24+ required**.

## Common Patterns

```go
// Pipeline
result := F.Pipe3(input, R.Map(transform), R.Chain(validate), R.Fold(onErr, onOk))

// Reusable pipeline
pipeline := F.Flow3(R.Map(normalize), R.Chain(validate), R.Map(format))

// Wrap Go error function
safeAtoi := R.Eitherize1(strconv.Atoi) // func(string) Result[int]

// Effect with DI
fetchUser := EFF.Eitherize(func(deps Deps, ctx context.Context) (*User, error) { ... })
val, err := EFF.RunSync(EFF.Provide[*User](myDeps)(fetchUser))(ctx)

// Optics
nameLens := L.MakeLens(func(p Person) string { return p.Name }, func(p Person, n string) Person { p.Name = n; return p })
updated := nameLens.Set("Bob")(person)
```

---

# Active Guidance Modes

## Mode 1: Auto-Detect

When the project imports `github.com/IBM/fp-go/v2`, silently apply these conventions:
- Use the import aliases from the table above
- Follow data-last patterns in all generated fp-go code
- Use proper type param ordering (non-inferrable first)
- Prefer idiomatic packages for new production code
- Use `F.Pipe` for immediate application, `F.Flow` for reusable pipelines
- Wrap Go `(value, error)` functions with `Eitherize`
- Use `Chain` instead of manual `if err != nil` when composing fp-go operations

## Mode 2: Monad Selection

When the user asks "which type should I use?" or is writing a new function:

1. **Ask**: Does the function perform I/O (network, disk, database)?
   - No → pure function, no monad needed (or `Result[A]` if it can fail)
   - Yes → continue
2. **Ask**: Can it fail with an error?
   - No → `IO[A]` (rare in practice)
   - Yes → continue
3. **Ask**: Does it need `context.Context` (cancellation, deadlines)?
   - No → `IOResult[A]`
   - Yes → continue
4. **Ask**: Does it need typed dependencies (DB, config, services)?
   - No → `ReaderIOResult[A]` via `context/readerioresult`
   - Yes → `Effect[C, A]` where C is the dependency struct
5. **Ask**: Is this a hot path (>1000 req/s)?
   - Yes → use idiomatic variant (`idiomatic/context/readerresult` or `idiomatic/ioresult`)
   - No → standard is fine

Present the recommendation with a concrete code skeleton.

## Mode 3: Code Review

When the user asks to review fp-go code, check for these anti-patterns:

| Anti-pattern | What to suggest |
|-------------|-----------------|
| Manual `if err != nil` between fp-go operations | Use `Chain` or `Pipe` for automatic error propagation |
| Using `Either[error, A]` instead of `Result[A]` | Use `Result[A]` unless custom error type needed |
| Passing `context.Context` explicitly through fp-go chains | Use `ReaderIOResult` or `Effect` — context threads automatically |
| Using standard packages in hot paths | Consider `idiomatic/` variants for 2-32x speedup |
| Data-first parameter order | Flip to data-last for composability |
| `struct{}` / `struct{}{}` | Use `F.Void` / `F.VOID` |
| `Compose` in pipelines | Use `Flow` (left-to-right) instead |
| Explicit type params when inference works | Remove unnecessary type annotations |
| Creating `func(A) Result[B]` manually from Go error functions | Use `result.Eitherize1(fn)` |
| Sequential `Chain` calls that don't depend on each other | Consider `Ap` for applicative parallelism |
| Missing `ChainEitherK` for pure validation | Lift pure `func(A) (B, error)` with `ChainEitherK` instead of wrapping in IO |
| Deeply nested `Chain(func(x) { return Chain(func(y) { ... }) })` | Use Do-notation (`Do`, `Bind`, `Let`) for readability |

## Mode 4: Migration

When the user wants to convert imperative Go to fp-go, follow this recipe:

1. **Identify effects**: Does the function do I/O? Handle errors? Need context? Need deps?
2. **Choose monad stack**: Use the decision tree from Mode 2
3. **Wrap external calls**: `result.Eitherize1(fn)` for each `(val, err)` function
4. **Replace error chains**: Convert `if err != nil` sequences to `F.Pipe` + `R.Chain`
5. **Extract pure logic**: Separate validation/transformation from I/O. Use `ChainEitherK` to lift pure functions.
6. **Build pipeline**: Compose with `F.Pipe` or `F.Flow`
7. **Add type aliases**: For readability: `type Result[A any] = result.Result[A]`

For detailed recipes, read `cookbook.md` from this skill's directory.

## Mode 5: Testing

When the project imports `github.com/franchb/fptest`, silently apply testing conventions:
- Use the fptest-go import aliases from the table above
- Prefer property-based tests over example-based for algebraic properties
- Use `assert.AssertSome`/`AssertRight`/`AssertOk`/`AssertEffect` instead of manual unwrapping
- Use `laws.*` for verifying typeclass laws on custom types

When the user asks "how should I test this?" or is writing tests for fp-go code:

1. **Custom type with algebraic operations?** (Monoid, Semigroup, Eq, Ord) → `laws.MonoidLaws`, `laws.EqLaws`, etc.
2. **Custom Functor/Monad/Applicative?** → `laws.FunctorLaws`, `laws.MonadLaws`, `laws.ApplicativeFullLaws`
3. **Optics (Lens/Prism)?** → `laws.LensLaws`
4. **Encode/Decode pair?** → `prop.RoundTrip` / `prop.RoundTripError`
5. **Optimized implementation?** → `prop.Oracle` against reference
6. **Idempotent transformation?** → `prop.Idempotent`
7. **Effectful pipeline?** → `mock.TrackedStub` + `assert.AssertIORight` / `assert.AssertEffect`
8. **Simple success/failure check?** → `assert.AssertSome`, `assert.AssertRight`, `assert.AssertOk`

When reviewing test code, check for these anti-patterns:

| Anti-pattern | What to suggest |
|-------------|-----------------|
| Manual `if opt.IsNone()` in tests | Use `assert.AssertSome(t, opt)` |
| `result.Unwrap()` without checking | Use `assert.AssertOk(t, r)` |
| Example-based tests for algebraic laws | Use `laws.FunctorLaws`, `laws.MonadLaws` with generators |
| Testing Map/Chain with one input | Property-based: generators + `laws.FunctorLaws` |
| Manual `effect.RunSync` + error check | Use `assert.AssertEffect(t, deps, ctx, eff)` |
| Missing law tests for custom Monoid | Add `laws.MonoidLaws` with generators |
| `reflect.DeepEqual` on fp-go types | Use `assert.AssertEq` with proper `Eq` instance |

For detailed testing patterns, read `testing.md` from this skill's directory.

---

# Companion File Loading

When deeper context is needed, read the appropriate companion file from this skill's directory:

| User need | File to read |
|-----------|-------------|
| "How do I X with fp-go?" / migration task | `cookbook.md` |
| Need type signatures or operation details | `core-patterns.md` |
| Advanced: Do-notation, DI, transformers, profunctors | `mastery.md` |
| "Does fp-go have X?" / specific API lookup | `full-reference.md` |
| Testing fp-go code, property-based testing, law verification | `testing.md` |

Use the `Read` tool to load the file. Only load what's needed for the current task — don't load all files at once.
