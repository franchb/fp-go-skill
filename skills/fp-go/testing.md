# fptest-go — Property-Based Testing for fp-go/v2

fptest-go (`github.com/franchb/fptest`) is a property-based testing library for Go that verifies algebraic laws in fp-go abstractions. It provides monadic generators, law verification, FP-aware assertions, higher-level property patterns, and functional mocking — all integrated with Go's standard `testing` package.

**Dependencies**: `github.com/IBM/fp-go/v2` + `pgregory.net/rapid`

## Import Conventions

| Alias | Package | Purpose |
|-------|---------|---------|
| `FPT`  | `github.com/franchb/fptest/gen` | Monadic generators |
| `FPTL` | `github.com/franchb/fptest/laws` | Algebraic law verification |
| `FPTA` | `github.com/franchb/fptest/assert` | FP-aware assertions |
| `FPTP` | `github.com/franchb/fptest/prop` | Property patterns |
| `FPTM` | `github.com/franchb/fptest/mock` | Functional mocking |

## Quick Start

```go
import (
    "testing"
    "pgregory.net/rapid"

    "github.com/IBM/fp-go/v2/option"
    "github.com/franchb/fptest/assert"
    "github.com/franchb/fptest/gen"
    "github.com/franchb/fptest/laws"
)

func TestOptionFunctorLaws(t *testing.T) {
    laws.FunctorLaws(t,
        gen.GenOption(rapid.Int()),           // Gen[Option[int]]
        gen.GenFunc[int](rapid.String()),     // Gen[func(int) string]
        gen.GenFunc[string](rapid.Float64()), // Gen[func(string) float64]
        option.Eq[int](func(a, b int) bool { return a == b }),
        option.Eq[float64](func(a, b float64) bool { return a == b }),
        option.Map[int, int],
        option.Map[int, string],
        option.Map[string, float64],
        option.Map[int, float64],
        func(a int) int { return a },
        func(f func(int) string, g func(string) float64) func(int) float64 {
            return func(a int) float64 { return g(f(a)) }
        },
    )
}
```

---

## Generators (`gen`)

### Core Type

```go
type Gen[A any] func(*rapid.T) A
```

`Gen[A]` is a monadic generator — a function from rapid's test state to a value of type `A`. Unlike rapid's opaque `*Generator[A]`, `Gen` supports `Map`, `Chain`, and `Ap` for composable, dependent test data generation.

### Core Operations

| Function | Signature | Purpose |
|----------|-----------|---------|
| `Of` | `func Of[A any](a A) Gen[A]` | Pure — always generates the same value |
| `Map` | `func Map[A, B any](ga Gen[A], f func(A) B) Gen[B]` | Transform generated values |
| `Chain` | `func Chain[A, B any](ga Gen[A], f func(A) Gen[B]) Gen[B]` | Dependent generation (value of B depends on generated A) |
| `Ap` | `func Ap[A, B any](gf Gen[func(A) B], ga Gen[A]) Gen[B]` | Applicative apply |
| `Filter` | `func Filter[A any](ga Gen[A], pred func(A) bool) Gen[A]` | Constrain output (may panic on repeated rejection) |
| `Map2` | `func Map2[A, B, C any](ga Gen[A], gb Gen[B], f func(A, B) C) Gen[C]` | Combine two generators |
| `Map3` | `func Map3[A, B, C, D any](ga Gen[A], gb Gen[B], gc Gen[C], f func(A, B, C) D) Gen[D]` | Combine three generators |
| `Pair` | `func Pair[A, B any](ga Gen[A], gb Gen[B]) Gen[[2]any]` | Generate pairs |
| `Slice` | `func Slice[A any](ga Gen[A], minLen, maxLen int) Gen[[]A]` | Generate variable-length slices |

### Rapid Interop

| Function | Signature | Purpose |
|----------|-----------|---------|
| `ToRapid` | `func ToRapid[A any](g Gen[A]) *rapid.Generator[A]` | Convert Gen to rapid generator |
| `FromRapid` | `func FromRapid[A any](g *rapid.Generator[A]) Gen[A]` | Convert rapid generator to Gen |
| `FromRapidLabeled` | `func FromRapidLabeled[A any](g *rapid.Generator[A], label string) Gen[A]` | Convert with label for shrinking |

### fp-go Type Generators

**Option generators:**

| Function | Signature | Notes |
|----------|-----------|-------|
| `GenOption` | `func GenOption[A any](genA *rapid.Generator[A]) *rapid.Generator[option.Option[A]]` | Random Some/None |
| `GenSome` | `func GenSome[A any](genA *rapid.Generator[A]) *rapid.Generator[option.Option[A]]` | Always Some |
| `GenNone` | `func GenNone[A any]() *rapid.Generator[option.Option[A]]` | Always None |
| `MonadicOption` | `func MonadicOption[A any](ga Gen[A]) Gen[option.Option[A]]` | Random Some/None via Gen monad |
| `MonadicSome` | `func MonadicSome[A any](ga Gen[A]) Gen[option.Option[A]]` | Always Some via Gen monad |

**Either generators:**

| Function | Signature | Notes |
|----------|-----------|-------|
| `GenEither` | `func GenEither[E, A any](genE *rapid.Generator[E], genA *rapid.Generator[A]) *rapid.Generator[either.Either[E, A]]` | Random Left/Right |
| `GenRight` | `func GenRight[E, A any](genA *rapid.Generator[A]) *rapid.Generator[either.Either[E, A]]` | Always Right |
| `GenLeft` | `func GenLeft[E, A any](genE *rapid.Generator[E]) *rapid.Generator[either.Either[E, A]]` | Always Left |
| `MonadicEither` | `func MonadicEither[E, A any](ge Gen[E], ga Gen[A]) Gen[either.Either[E, A]]` | Random Left/Right via Gen |
| `MonadicRight` | `func MonadicRight[E, A any](ga Gen[A]) Gen[either.Either[E, A]]` | Always Right via Gen |
| `MonadicLeft` | `func MonadicLeft[E, A any](ge Gen[E]) Gen[either.Either[E, A]]` | Always Left via Gen |

**IO generators:**

| Function | Signature | Notes |
|----------|-----------|-------|
| `GenIO` | `func GenIO[A any](genA *rapid.Generator[A]) *rapid.Generator[io.IO[A]]` | IO wrapping generated value |
| `GenIOEither` | `func GenIOEither[E, A any](genE *rapid.Generator[E], genA *rapid.Generator[A]) *rapid.Generator[ioeither.IOEither[E, A]]` | Random success/failure |
| `GenIORight` | `func GenIORight[E, A any](genA *rapid.Generator[A]) *rapid.Generator[ioeither.IOEither[E, A]]` | Always succeeds |
| `GenIOLeft` | `func GenIOLeft[E, A any](genE *rapid.Generator[E]) *rapid.Generator[ioeither.IOEither[E, A]]` | Always fails |
| `MonadicIO` | `func MonadicIO[A any](ga Gen[A]) Gen[io.IO[A]]` | IO via Gen monad |

**Function generators:**

| Function | Signature | Notes |
|----------|-----------|-------|
| `GenFunc` | `func GenFunc[A, B any](genB *rapid.Generator[B]) *rapid.Generator[func(A) B]` | Constant function (ignores input) |
| `GenEndomorphism` | `func GenEndomorphism[A any](genA *rapid.Generator[A]) *rapid.Generator[func(A) A]` | A → A function |
| `MonadicFunc` | `func MonadicFunc[A, B any](gb Gen[B]) Gen[func(A) B]` | Constant function via Gen |

### Generator Composition Patterns

**Independent combination with Map3:**
```go
type Person struct { Name string; Age int; Email string }

genPerson := gen.Map3(
    gen.FromRapid(rapid.StringMatching(`[a-z]{3,10}`)),
    gen.FromRapid(rapid.IntRange(18, 120)),
    gen.FromRapid(rapid.StringMatching(`[a-z]+@[a-z]+\.[a-z]{2,3}`)),
    func(name string, age int, email string) Person {
        return Person{Name: name, Age: age, Email: email}
    },
)
```

**Dependent generation with Chain:**
```go
type Order struct { Items []string; Total float64 }

genOrder := gen.Chain(
    gen.Slice(gen.FromRapid(rapid.StringMatching(`item-[0-9]{3}`)), 1, 5),
    func(items []string) gen.Gen[Order] {
        return gen.Map(
            gen.FromRapid(rapid.Float64Range(1.0, 100.0*float64(len(items)))),
            func(total float64) Order { return Order{Items: items, Total: total} },
        )
    },
)
```

**Filtered generation:**
```go
genAdult := gen.Filter(genPerson, func(p Person) bool { return p.Age >= 18 })
```

**Converting to rapid for law tests:**
```go
rapidGen := gen.ToRapid(genPerson) // *rapid.Generator[Person]
```

---

## Law Verification (`laws`)

### Typeclass Interfaces

```go
type Pointed[A, FA any] interface {
    Of(A) FA
}

type Functor[A, B, FA, FB any] interface {
    Map(func(A) B) func(FA) FB
}

type Apply[A, B, FA, FB, FAB any] interface {
    Functor[A, B, FA, FB]
    Ap(FA) func(FAB) FB
}

type Applicative[A, B, FA, FB, FAB any] interface {
    Apply[A, B, FA, FB, FAB]
    Pointed[A, FA]
}

type Chainable[A, B, FA, FB any] interface {
    Chain(func(A) FB) func(FA) FB
}

type Monad[A, B, FA, FB, FAB any] interface {
    Applicative[A, B, FA, FB, FAB]
    Chainable[A, B, FA, FB]
}
```

Constructors: `MakePointed`, `MakeFunctor`, `MakeApply`, `MakeApplicative`, `MakeChainable`, `MakeMonad`.

### Law Functions

#### Functor Laws

```go
func FunctorLaws[FA, FB, FC, A, B, C any](
    t *testing.T,
    genFA *rapid.Generator[FA],
    genAB *rapid.Generator[func(A) B],
    genBC *rapid.Generator[func(B) C],
    eqFA func(FA, FA) bool,
    eqFC func(FC, FC) bool,
    fmapAA func(func(A) A) func(FA) FA,
    fmapAB func(func(A) B) func(FA) FB,
    fmapBC func(func(B) C) func(FB) FC,
    fmapAC func(func(A) C) func(FA) FC,
    identity func(A) A,
    compose func(func(A) B, func(B) C) func(A) C,
)
```

Verifies:
- **Identity**: `Map(id)(fa) == fa`
- **Composition**: `Map(g . f)(fa) == Map(g)(Map(f)(fa))`

#### Applicative Laws

```go
func ApplicativeLaws[FA, FB, FAB, A, B any](
    t *testing.T,
    genA *rapid.Generator[A],
    genFA *rapid.Generator[FA],
    genAB *rapid.Generator[func(A) B],
    eqFA func(FA, FA) bool,
    eqFB func(FB, FB) bool,
    ofA func(A) FA,
    ofB func(B) FB,
    ofAB func(func(A) B) FAB,
    fmapAA func(func(A) A) func(FA) FA,
    apAB func(FA) func(FAB) FB,
    identity func(A) A,
)
```

Verifies: **Identity** (`Ap(Of(id))(fa) == fa`) and **Homomorphism** (`Ap(Of(f))(Of(a)) == Of(f(a))`).

Additional: `ApplicativeInterchange`, `ApplyAssociativeComposition`.

**Full Applicative laws with pre-wired instances:**

```go
func ApplicativeFullLaws[FA, FB, FC, FAB, FBC, FAC, FABAC, FABB, A, B, C any](
    t *testing.T,
    genFA *rapid.Generator[FA],
    genA *rapid.Generator[A],
    genAB *rapid.Generator[func(A) B],
    genBC *rapid.Generator[func(B) C],
    inst *ApplicativeInstances[FA, FB, FC, FAB, FBC, FAC, FABAC, FABB, A, B, C],
)
```

Use with `OptionApplicativeInstances` or `EitherApplicativeInstances` for zero-boilerplate law verification.

#### Monad Laws

```go
func MonadLaws[FA, FB, FC, A, B, C any](
    t *testing.T,
    genA *rapid.Generator[A],
    genFA *rapid.Generator[FA],
    genKleisliAB *rapid.Generator[func(A) FB],
    genKleisliBC *rapid.Generator[func(B) FC],
    eqFB func(FB, FB) bool,
    eqFA func(FA, FA) bool,
    eqFC func(FC, FC) bool,
    of func(A) FA,
    chainAB func(func(A) FB) func(FA) FB,
    chainBC func(func(B) FC) func(FB) FC,
    chainAC func(func(A) FC) func(FA) FC,
)
```

Verifies:
- **Left identity**: `Chain(f)(Of(a)) == f(a)`
- **Right identity**: `Chain(Of)(fa) == fa` (via `MonadLawsFull`)
- **Associativity**: `Chain(g)(Chain(f)(fa)) == Chain(x => Chain(g)(f(x)))(fa)`

Additional: `ChainAssociativity` for just the associativity law.

#### Algebraic Structure Laws

**Semigroup** — verifies associativity:
```go
func SemigroupLaws[A any](t *testing.T, genA *rapid.Generator[A], eqA func(A, A) bool, concat func(A, A) A)
func SemigroupInterfaceLaws[A any](t *testing.T, genA *rapid.Generator[A], eqA func(A, A) bool, sg interface{ Concat(A, A) A })
```

**Monoid** — verifies associativity + identity (calls `SemigroupLaws` internally):
```go
func MonoidLaws[A any](t *testing.T, genA *rapid.Generator[A], eqA func(A, A) bool, concat func(A, A) A, empty A)
func MonoidInterfaceLaws[A any](t *testing.T, genA *rapid.Generator[A], eqA func(A, A) bool, m interface{ Concat(A, A) A; Empty() A })
```

**Eq** — verifies reflexivity, symmetry, transitivity:
```go
func EqLaws[A any](t *testing.T, genA *rapid.Generator[A], equals func(A, A) bool)
func EqInterfaceLaws[A any](t *testing.T, genA *rapid.Generator[A], eqA interface{ Equals(A, A) bool })
```

**Ord** — verifies antisymmetry, transitivity, totality, consistency (calls `EqLaws` internally):
```go
func OrdLaws[A any](t *testing.T, genA *rapid.Generator[A], equals func(A, A) bool, compare func(A, A) int)
func OrdInterfaceLaws[A any](t *testing.T, genA *rapid.Generator[A], ordA interface{ Equals(A, A) bool; Compare(A, A) int })
```

#### Lens Laws

```go
func LensLaws[S, A any](
    t *testing.T,
    genS *rapid.Generator[S],
    genA *rapid.Generator[A],
    eqS func(S, S) bool,
    eqA func(A, A) bool,
    get func(S) A,
    set func(A) func(S) S,
)
```

Verifies:
- **Get-Set**: `Set(Get(s))(s) == s`
- **Set-Get**: `Get(Set(a)(s)) == a`
- **Set-Set**: `Set(b)(Set(a)(s)) == Set(b)(s)`

### Pre-Wired Instances

For Option and Either, all typeclass operations are pre-wired:

```go
func OptionApplicativeInstances[A, B, C comparable]() *ApplicativeInstances[...]
func EitherApplicativeInstances[E, A, B, C comparable]() *ApplicativeInstances[...]
```

Usage:
```go
func TestOptionApplicativeFullLaws(t *testing.T) {
    inst := laws.OptionApplicativeInstances[int, string, float64]()
    laws.ApplicativeFullLaws(t,
        gen.GenOption(rapid.Int()),
        rapid.Int(),
        gen.GenFunc[int](rapid.String()),
        gen.GenFunc[string](rapid.Float64()),
        inst,
    )
}
```

---

## Assertions (`assert`)

FP-aware assertions that unwrap fp-go types and fail the test on unexpected variants. All assertion functions return the unwrapped value, enabling chaining.

### Option Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertSome` | `func AssertSome[A any](t testing.TB, o option.Option[A]) A` | None |
| `AssertNone` | `func AssertNone[A any](t testing.TB, o option.Option[A])` | Some |
| `AssertSomeEq` | `func AssertSomeEq[A comparable](t testing.TB, o option.Option[A], want A)` | None or wrong value |

### Either Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertRight` | `func AssertRight[E, A any](t testing.TB, e either.Either[E, A]) A` | Left |
| `AssertLeft` | `func AssertLeft[E, A any](t testing.TB, e either.Either[E, A]) E` | Right |
| `AssertRightEq` | `func AssertRightEq[E any, A comparable](t testing.TB, e either.Either[E, A], want A)` | Left or wrong value |
| `AssertLeftEq` | `func AssertLeftEq[E comparable, A any](t testing.TB, e either.Either[E, A], want E)` | Right or wrong value |

### Result Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertOk` | `func AssertOk[A any](t testing.TB, r result.Result[A]) A` | Error |
| `AssertErr` | `func AssertErr[A any](t testing.TB, r result.Result[A]) error` | Ok |
| `AssertOkEq` | `func AssertOkEq[A comparable](t testing.TB, r result.Result[A], want A)` | Error or wrong value |

### IO Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertIO` | `func AssertIO[A any](t testing.TB, action io.IO[A]) A` | — (executes and returns) |
| `AssertIORight` | `func AssertIORight[E, A any](t testing.TB, action ioeither.IOEither[E, A]) A` | Left after execution |
| `AssertIOLeft` | `func AssertIOLeft[E, A any](t testing.TB, action ioeither.IOEither[E, A]) E` | Right after execution |
| `AssertIOSome` | `func AssertIOSome[A any](t testing.TB, action io.IO[option.Option[A]]) A` | None after execution |
| `AssertIONone` | `func AssertIONone[A any](t testing.TB, action io.IO[option.Option[A]])` | Some after execution |
| `AssertIOEq` | `func AssertIOEq[A comparable](t testing.TB, action io.IO[A], want A)` | Wrong value after execution |
| `AssertIOEitherEq` | `func AssertIOEitherEq[E, A comparable](t testing.TB, action ioeither.IOEither[E, A], want either.Either[E, A], eqEither func(...) bool)` | Mismatch after execution |

### Effect Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertEffect` | `func AssertEffect[C, A any](t testing.TB, deps C, ctx context.Context, eff effect.Effect[C, A]) A` | Error after execution |
| `AssertEffectErr` | `func AssertEffectErr[C, A any](t testing.TB, deps C, ctx context.Context, eff effect.Effect[C, A]) error` | Success after execution |
| `AssertEffectEq` | `func AssertEffectEq[C, A comparable](t testing.TB, deps C, ctx context.Context, eff effect.Effect[C, A], want A)` | Error or wrong value |

### Eq-Based Assertions

| Function | Signature | Fails when |
|----------|-----------|------------|
| `AssertEq` | `func AssertEq[A any](t testing.TB, eqA eq.Eq[A], got, want A)` | Not equal per Eq instance |
| `AssertEqFunc` | `func AssertEqFunc[A any](t testing.TB, equals func(A, A) bool, got, want A)` | Not equal per function |
| `AssertNotEq` | `func AssertNotEq[A any](t testing.TB, eqA eq.Eq[A], got, notWant A)` | Equal when shouldn't be |

### Assertion Chaining

Assertions return unwrapped values, enabling pipeline-style test verification:

```go
// Chain: execute IOEither, unwrap Right, then unwrap inner Option
user := assert.AssertSome(t, assert.AssertIORight[error](t, fetchUser(ctx)))

// Execute Effect, get result
val := assert.AssertEffect(t, deps, ctx, myEffect)

// Verify both success and value in one line
assert.AssertOkEq(t, result, 42)
```

---

## Property Patterns (`prop`)

Higher-level property verification patterns built on rapid.

### Round-Trip (Codec Verification)

Verifies `decode(encode(a)) == a` for all generated `a`.

```go
func RoundTrip[A, B any](t *testing.T, name string, genA *rapid.Generator[A], eqA func(A, A) bool, encode func(A) B, decode func(B) A)
func RoundTripPartial[A, B any](t *testing.T, name string, genA *rapid.Generator[A], eqA func(A, A) bool, encode func(A) B, decode func(B) (A, bool))
func RoundTripError[A, B any](t *testing.T, name string, genA *rapid.Generator[A], eqA func(A, A) bool, encode func(A) B, decode func(B) (A, error))
```

Three variants for different decode signatures:
- `RoundTrip` — pure decode
- `RoundTripPartial` — decode returns `(A, bool)`
- `RoundTripError` — decode returns `(A, error)`

```go
// JSON round-trip
prop.RoundTripError(t, "User/JSON", genUser, eqUser,
    func(u User) []byte { b, _ := json.Marshal(u); return b },
    func(b []byte) (User, error) { var u User; err := json.Unmarshal(b, &u); return u, err },
)

// Int parsing (partial)
prop.RoundTripPartial(t, "int/string", rapid.Int(),
    func(a, b int) bool { return a == b },
    strconv.Itoa,
    func(s string) (int, bool) { v, err := strconv.Atoi(s); return v, err == nil },
)
```

### Oracle (Reference Implementation)

Verifies `impl(a) == reference(a)` for all generated `a`. Use when you have an optimized implementation and a simple reference.

```go
func Oracle[A, B any](t *testing.T, name string, genA *rapid.Generator[A], eqB func(B, B) bool, impl func(A) B, reference func(A) B)
```

```go
prop.Oracle(t, "validateAge", rapid.IntRange(-1000, 1000),
    func(a, b bool) bool { return a == b },
    validateAgeOptimized,
    validateAgeReference,
)
```

### Idempotent

Verifies `f(f(x)) == f(x)` — applying the function twice gives the same result as once.

```go
func Idempotent[A any](t *testing.T, name string, genA *rapid.Generator[A], eqA func(A, A) bool, f func(A) A)
```

```go
prop.Idempotent(t, "normalizeEmail", rapid.StringMatching(`[A-Za-z]+@[a-z]+\.[a-z]{2,3}`),
    func(a, b string) bool { return a == b },
    strings.ToLower,
)
```

### Commutative

Verifies `f(a, b) == f(b, a)` — order of arguments doesn't matter.

```go
func Commutative[A, B any](t *testing.T, name string, genA *rapid.Generator[A], eqB func(B, B) bool, f func(A, A) B)
```

```go
prop.Commutative(t, "max", rapid.Int(),
    func(a, b int) bool { return a == b },
    max,
)
```

### Invariant

Verifies that a predicate holds for all generated values.

```go
func Invariant[A any](t *testing.T, name string, genA *rapid.Generator[A], predicate func(A) bool)
```

```go
prop.Invariant(t, "validUsername", genValidUsername,
    func(u string) bool { return len(u) >= 3 && len(u) <= 20 },
)
```

---

## Mocking (`mock`)

Functional mocking utilities that maintain referential transparency at the type level.

### IORef — Mutable Reference Cell

Thread-safe mutable reference in the IO monad. All mutations are wrapped in `IO`, preserving the type-level guarantee that side effects are tracked.

```go
type IORef[A any] struct { /* sync.Mutex + val */ }

func NewIORef[A any](initial A) io.IO[*IORef[A]]
func (r *IORef[A]) Read() io.IO[A]
func (r *IORef[A]) Write(a A) io.IO[struct{}]
func (r *IORef[A]) Modify(f func(A) A) io.IO[struct{}]
func (r *IORef[A]) ReadUnsafe() A  // Escape hatch for assertions
```

```go
ref := mock.NewIORef(0)()
ref.Write(42)()
val := ref.Read()() // 42
ref.Modify(func(n int) int { return n + 1 })()
val = ref.Read()() // 43
```

### CallTracker

Records method calls for verification. Built on `IORef` for thread safety.

```go
type Call struct {
    Method string
    Args   []any
}

type CallTracker struct { /* IORef[[]Call] */ }

func NewCallTracker() io.IO[*CallTracker]
func (ct *CallTracker) Record(method string, args ...any) io.IO[struct{}]
func (ct *CallTracker) RecordSync(method string, args ...any)
func (ct *CallTracker) Calls() io.IO[[]Call]
func (ct *CallTracker) CallsUnsafe() []Call  // Escape hatch
func (ct *CallTracker) CallCount(method string) int
func (ct *CallTracker) CallsFor(method string) []Call
```

### Stub Builders

Create mock implementations for `func(A) IOEither[E, R]` signatures.

| Function | Signature | Behavior |
|----------|-----------|----------|
| `Stub` | `func Stub[A, E, R any](value R) func(A) ioeither.IOEither[E, R]` | Always succeeds |
| `StubError` | `func StubError[A, E, R any](err E) func(A) ioeither.IOEither[E, R]` | Always fails |
| `TrackedStub` | `func TrackedStub[A, E, R any](ct *CallTracker, method string, value R) func(A) ioeither.IOEither[E, R]` | Succeeds + records call |
| `TrackedStubError` | `func TrackedStubError[A, E, R any](ct *CallTracker, method string, err E) func(A) ioeither.IOEither[E, R]` | Fails + records call |
| `TrackedFunc` | `func TrackedFunc[A, E, R any](ct *CallTracker, method string, impl func(A) ioeither.IOEither[E, R]) func(A) ioeither.IOEither[E, R]` | Delegates + records call |

```go
ct := mock.NewCallTracker()()

// Create stubs
findUser := mock.TrackedStub[string, error, User](ct, "FindUser", User{Name: "Alice"})
saveUser := mock.TrackedStub[User, error, struct{}](ct, "SaveUser", struct{}{})

// Use in pipeline
result := findUser("alice-id")()
user := assert.AssertIORight[error](t, result)

// Verify calls
assert.AssertEqFunc(t, func(a, b int) bool { return a == b }, ct.CallCount("FindUser"), 1)
calls := ct.CallsFor("FindUser")
assert.AssertEqFunc(t, func(a, b string) bool { return a == b }, calls[0].Args[0].(string), "alice-id")
```

---

## Complete Recipes

### Recipe 1: Testing a Custom Monoid

Verify that a domain type's `Concat` and `Empty` operations satisfy monoid laws.

```go
// Domain type: Money with same-currency addition
type Money struct {
    Amount   int
    Currency string
}

func moneyConcat(a, b Money) Money {
    if a.Currency != b.Currency { return a } // same-currency only
    return Money{Amount: a.Amount + b.Amount, Currency: a.Currency}
}

var moneyEmpty = Money{Amount: 0, Currency: "USD"}

func genMoney() *rapid.Generator[Money] {
    return rapid.Custom(func(t *rapid.T) Money {
        return Money{
            Amount:   rapid.IntRange(0, 10000).Draw(t, "amount"),
            Currency: "USD", // fix currency for monoid laws
        }
    })
}

func TestMoneyMonoid(t *testing.T) {
    eqMoney := func(a, b Money) bool { return a == b }
    laws.MonoidLaws(t, genMoney(), eqMoney, moneyConcat, moneyEmpty)
}
```

### Recipe 2: Testing a ReaderIOResult Pipeline

Mock dependencies, run the pipeline, and assert results.

```go
type UserRepo interface {
    FindByID(id string) ioeither.IOEither[error, User]
}

type mockRepo struct {
    tracker *mock.CallTracker
    findFn  func(string) ioeither.IOEither[error, User]
}

func (m *mockRepo) FindByID(id string) ioeither.IOEither[error, User] {
    return m.findFn(id)
}

func TestGetUserPipeline(t *testing.T) {
    ct := mock.NewCallTracker()()
    repo := &mockRepo{
        tracker: ct,
        findFn:  mock.TrackedStub[string, error, User](ct, "FindByID", User{Name: "Alice"}),
    }

    result := getUserPipeline(repo, "alice-id")()
    user := assert.AssertRight[error](t, result)
    assert.AssertEqFunc(t, func(a, b string) bool { return a == b }, user.Name, "Alice")
    assert.AssertEqFunc(t, func(a, b int) bool { return a == b }, ct.CallCount("FindByID"), 1)
}
```

### Recipe 3: Testing an Effect Service

```go
type Deps struct {
    DB     DBClient
    Logger Logger
}

func TestFetchUser(t *testing.T) {
    ct := mock.NewCallTracker()()
    deps := Deps{
        DB:     &mockDB{findFn: mock.TrackedStub[string, error, User](ct, "Find", User{Name: "Bob"})},
        Logger: &mockLogger{},
    }

    user := assert.AssertEffect(t, deps, context.Background(), fetchUserEffect("bob-id"))
    assert.AssertEqFunc(t, func(a, b string) bool { return a == b }, user.Name, "Bob")
    assert.AssertEqFunc(t, func(a, b int) bool { return a == b }, ct.CallCount("Find"), 1)
}
```

### Recipe 4: Verifying Lens Laws for Nested Config

```go
type AppConfig struct {
    Server ServerConfig
}

type ServerConfig struct {
    Host string
    Port int
}

func TestServerPortLens(t *testing.T) {
    genConfig := rapid.Custom(func(t *rapid.T) AppConfig {
        return AppConfig{Server: ServerConfig{
            Host: rapid.StringMatching(`[a-z]+\.example\.com`).Draw(t, "host"),
            Port: rapid.IntRange(1024, 65535).Draw(t, "port"),
        }}
    })

    laws.LensLaws(t,
        genConfig,
        rapid.IntRange(1024, 65535),
        func(a, b AppConfig) bool { return a == b },
        func(a, b int) bool { return a == b },
        func(c AppConfig) int { return c.Server.Port },
        func(port int) func(AppConfig) AppConfig {
            return func(c AppConfig) AppConfig {
                c.Server.Port = port
                return c
            }
        },
    )
}
```

### Recipe 5: Round-Trip Testing a JSON Codec

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}

func TestUserJSONRoundTrip(t *testing.T) {
    genUser := rapid.Custom(func(t *rapid.T) User {
        return User{
            Name:  rapid.StringMatching(`[A-Za-z ]{1,50}`).Draw(t, "name"),
            Email: rapid.StringMatching(`[a-z]+@[a-z]+\.[a-z]{2,3}`).Draw(t, "email"),
            Age:   rapid.IntRange(0, 150).Draw(t, "age"),
        }
    })

    prop.RoundTripError(t, "User/JSON", genUser,
        func(a, b User) bool { return a == b },
        func(u User) []byte { b, _ := json.Marshal(u); return b },
        func(b []byte) (User, error) {
            var u User
            err := json.Unmarshal(b, &u)
            return u, err
        },
    )
}
```

---

## Test Strategy Decision Tree

When deciding how to test fp-go code, follow this order:

1. **Custom type with algebraic operations?** (Monoid, Semigroup, Eq, Ord)
   → Verify laws with `laws.MonoidLaws`, `laws.EqLaws`, etc.

2. **Custom Functor/Monad/Applicative?**
   → Verify with `laws.FunctorLaws`, `laws.MonadLaws`, `laws.ApplicativeFullLaws`

3. **Optics (Lens/Prism)?**
   → Verify with `laws.LensLaws`

4. **Encode/Decode pair?**
   → Verify with `prop.RoundTrip` / `prop.RoundTripError`

5. **Optimized implementation with known-correct reference?**
   → Compare with `prop.Oracle`

6. **Transformation that should be idempotent?** (normalization, formatting)
   → Verify with `prop.Idempotent`

7. **Binary operation that should be order-independent?**
   → Verify with `prop.Commutative`

8. **Effectful pipeline (IOEither, ReaderIOResult, Effect)?**
   → Mock with `mock.TrackedStub`, assert with `assert.AssertIORight` / `assert.AssertEffect`

9. **Simple success/failure check on fp-go types?**
   → Use `assert.AssertSome`, `assert.AssertRight`, `assert.AssertOk`, etc.
