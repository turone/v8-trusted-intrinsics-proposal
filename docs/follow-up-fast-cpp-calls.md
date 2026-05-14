# Follow-up Proposal: Fast Internal C++ Calls for Node.js Builtins

## Status

- Future follow-up proposal
- Not part of the Trusted Initial Intrinsics MVP
- Should be considered only after the main PoC produces benchmark data
- Requires independent profiling, benchmarking, and review

---

## Problem

Node.js internal JavaScript often calls small C++ helpers through `internalBinding()`. For tiny functions, the JS→C++ boundary cost can dominate the actual work.

Ordinary C++ callbacks go through a generic callback path (`v8::FunctionCallbackInfo<v8::Value>`), which involves:
- argument unpacking from the V8 heap
- return value boxing
- full V8 API overhead

V8 Fast API callbacks allow selected C++ embedder functions to be called more directly from optimized code when argument types match. This can potentially reduce overhead for tiny, boundary-dominated calls.

**Do not claim guaranteed 10× speedup.** Benefit is highly dependent on call-site frequency, argument type stability, and whether the function is boundary-dominated. Subject to benchmarks.

---

## Goal

- Identify small hot Node internal C++ bindings where call overhead is significant relative to the work performed.
- Add Fast API variants alongside existing slow callbacks.
- Preserve full existing semantics by keeping slow fallback paths.
- Use benchmarks to decide which functions are worth converting.

---

## Non-goals

- Do not replace large I/O operations (libuv-dominated work).
- Do not fast-path functions where C++ work dominates call overhead.
- Do not change public Node.js API semantics.
- Do not remove slow callbacks — they remain the semantic source of truth.
- Do not require all internal bindings to use Fast API.
- Do not merge with the Trusted Initial Intrinsics MVP.

---

## Relationship to Trusted Initial Intrinsics

| Proposal | Layer optimized |
|---|---|
| Trusted Initial Intrinsics | Internal JS → JS builtins (`arr.push`, `Object.keys`) |
| Fast Internal C++ Calls (this doc) | Internal JS → C++ bindings (`internalBinding(...).foo(...)`) |

These are complementary and independent. Neither depends on the other. They should be benchmarked separately.

---

## How V8 Fast API Helps

A normal JS→C++ binding call pays full generic callback overhead on every call, regardless of optimization tier.

V8 Fast API allows optimized code (Maglev/TurboFan) to call a C++ function via a typed fast path when argument types match the declared signature. If arguments do not match, or the function requests fallback, execution routes through the slow callback.

```cpp
// Slow callback: full V8 API path — always present, always correct.
void SlowFoo(const v8::FunctionCallbackInfo<v8::Value>& args);

// Fast callback: typed fast path — called only from optimized code.
int32_t FastFoo(int32_t value, v8::FastApiCallbackOptions& options) {
  if (value < 0) {
    options.fallback = true;  // route to SlowFoo
    return 0;
  }
  return value + 1;
}
```

JavaScript call site:
```js
const { foo } = internalBinding('example');
foo(123);  // may use fast path when optimized
```

### Registration sketch

Fast API callbacks are registered through `v8::CFunction` and `v8::FunctionTemplate`. The exact API depends on the V8 version embedded by Node — always audit `v8-fast-api-calls.h` against the target V8 tag before implementation.

Single fast signature:
```cpp
static const v8::CFunction fast_foo =
    v8::CFunction::MakeWithFallbackSupport(FastFoo);

// Registration shape is illustrative.
auto tmpl = v8::FunctionTemplate::New(
    isolate,
    SlowFoo,
    v8::Local<v8::Value>(),
    v8::Local<v8::Signature>(),
    1,
    v8::ConstructorBehavior::kThrow,
    v8::SideEffectType::kHasSideEffect,
    &fast_foo);
```

Multiple overloads (different argument types):
```cpp
v8::CFunction overloads[] = {
  v8::CFunction::Make(FastFooInt),
  v8::CFunction::Make(FastFooDouble),
};

auto tmpl = v8::FunctionTemplate::NewWithCFunctionOverloads(
    isolate,
    SlowFoo,
    v8::Local<v8::Value>(),
    v8::Local<v8::Signature>(),
    1,
    v8::ConstructorBehavior::kThrow,
    v8::SideEffectType::kHasSideEffect,
    v8::MemorySpan<const v8::CFunction>(overloads));
```

Key constraints:
- Fast callbacks must always have a slow callback fallback.
- Slow callback is the semantic source of truth; fast path is an optimization only.
- Exact API shape depends on the Node/V8 version; audit before implementation.

### Tiering and fallback behavior

Fast API is an optimization path, not a semantic mode.

- Fast calls are only used when V8 has optimized code (Maglev/TurboFan) capable of using the fast callback.
- Before optimization, under `--jitless`, or when argument types do not match the fast signature, execution always uses the slow callback.
- After deopt, calls revert to the slow path.
- **Correctness must be entirely provided by the slow callback.** The fast path must produce identical results, but must never be the only correct implementation.
- Benchmarks must measure both the optimized fast-path performance and slow-path/fallback behavior.

---

## Candidate Selection Criteria

### Good candidates

- Very small C++ helper where boundary cost is measurable
- Hot call site (confirmed by profiling)
- Stable argument types (low polymorphism)
- Primitive argument and return types (`int32_t`, `double`, `bool`)
- Little or no allocation
- No complex JS re-entry
- Easy slow fallback path

### Poor candidates

- Heavy I/O or libuv-dominated work
- Crypto/compression work
- Functions dominated by OS calls
- Functions returning complex JS objects
- Functions with many polymorphic argument shapes
- Functions where fallback would happen frequently
- Functions where slow path is already negligible

---

## MVP Candidate

Phase 0 should first audit `internalBinding()` call sites via profiling — do not guess.

Steps:
1. Profile Node.js under a representative workload.
2. Identify the highest-call-frequency small C++ binding functions.
3. Check argument type stability for the top candidates.
4. Pick one function with simple primitive arguments and return value and a measurable hot call site.
5. Add fast variant and slow fallback.
6. Measure before/after.

Checklist:
- [ ] Identify candidate binding via profiling
- [ ] Confirm hot call site
- [ ] Confirm simple, stable argument types
- [ ] Audit V8 Fast API registration API against target V8 version
- [ ] Add Fast API callback
- [ ] Keep slow fallback as semantic source of truth
- [ ] Add tests covering both fast and slow paths
- [ ] Add benchmark (see below)
- [ ] Measure fallback rate

---

## Benchmark Plan

- Compare slow callback vs Fast API callback for the candidate function.
- Include warmup iterations to trigger optimization before measuring the fast path.
- Run with default V8 flags (to measure fast path).
- Run with `--jitless` or equivalent to confirm slow-path correctness and measure slow-path overhead.
- Measure fallback rate (how often the fast path is bypassed due to type mismatch).
- Measure deopt count if available.
- Measure end-to-end Node benchmark only after microbenchmark confirms benefit at the boundary level.

| Benchmark | Slow callback | Fast API | Delta | Notes |
|---|---|---|---|---|
| candidate microbench (warm) | TBD | TBD | TBD | fast path |
| candidate microbench (`--jitless`) | TBD | TBD | TBD | slow path |
| fallback case (type mismatch) | TBD | TBD | TBD | fallback rate |
| Node macro-benchmark | TBD | TBD | TBD | end-to-end |

---

## Correctness Requirements

- Slow and fast paths must produce identical results for all valid inputs.
- Fallback must not duplicate visible side effects.
- Unsupported argument types must behave like the current slow implementation.
- Error behavior must match the current implementation.
- Tests must cover:
  - fast path (optimized, matching argument types)
  - slow fallback (type mismatch or `options.fallback = true`)
  - `--jitless` (always slow path)

---

## Risks

- **Tiering dependency:** Under `--jitless`, before optimization, or after deopt, calls use the slow path. Fast API is a performance optimization, not a semantic guarantee.
- **Deopt/fallback risk:** Type instability at the call site can cause frequent fallback, erasing performance gains.
- **Benchmarking risk:** Microbenchmarks must warm up enough to reach the optimized tier; must separately measure fallback behavior.
- **Fast API surface stability:** Fast API internal shape may change across V8 versions.
- **More C++ signatures to maintain:** Each fast-path function adds a typed signature alongside the slow callback.
- **Wrong candidate selection:** If the chosen function is not actually boundary-dominated, no measurable benefit will appear.
- **API design awkwardness:** Type restrictions can force awkward interface decisions.

Do not proceed without profiling data identifying a real boundary-dominated hotspot.

---

## Recommended Rollout

1. Profile Node.js under representative workloads. Identify boundary-dominated bindings.
2. Pick one tiny candidate.
3. Implement Fast API variant; keep slow fallback as the semantic source of truth.
4. Benchmark (see Benchmark Plan).
5. If benefit is measurable, repeat for a small allowlist.
6. Stop if benefits are not measurable in end-to-end Node benchmarks.

**Do not proceed without benchmark evidence.**

---

## Key Contacts / Review Areas

Review areas (not prescriptive assignments):
- V8 Fast API / API reviewers (`v8-fast-api-calls.h` surface)
- V8 compiler team for optimized fast-call lowering (Maglev/TurboFan)
- Node.js performance team
- Node.js subsystem owners for the selected binding
- Security/correctness reviewers if the candidate binding processes user-provided data

Likely relevant stakeholders (subject to availability):
- Node/V8 bridge maintainers (e.g. Joyee Cheung)
- Node performance maintainers (e.g. Yagiz Nizipli)
- V8 Fast API maintainers
