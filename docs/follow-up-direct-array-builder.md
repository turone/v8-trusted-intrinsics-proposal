# Follow-up Proposal: Direct Array Builder API for V8 Embedders

## Status

- Future independent follow-up proposal
- Not part of the Trusted Initial Intrinsics MVP
- Not a prerequisite for Fast Internal C++ Calls
- Should be considered only after a concrete Node C++ array-building hotspot is identified and profiled

---

## Problem

Node.js C++ code often constructs JS arrays to return results to JavaScript. The typical pattern is:

```cpp
auto result = v8::Array::New(isolate, count);
for (size_t i = 0; i < count; i++) {
  result->Set(context, i, values[i]).Check();
}
return result;
```

The `array->Set(context, index, value)` path may pay more overhead than necessary for internally created ordinary arrays, because it goes through the generic property assignment machinery including:
- handle checks and context validation
- potential prototype/descriptor lookup
- write barriers for GC

For C++ code that builds arrays of known size or appends many values from internal data, a more direct construction path could reduce overhead.

---

## Goal

- Prototype or propose a safe direct array-building helper for embedder use.
- Optimize C++ code that creates ordinary JS arrays from internally known values.
- Preserve normal JS semantics for the resulting arrays after construction.
- Avoid unnecessary property machinery during internal construction.

---

## Non-goals

- Do not change JavaScript Array semantics.
- Do not affect userland array operations.
- Do not bypass GC write barriers.
- Do not work on arbitrary user-provided arrays.
- Do not mutate arrays that may already be observable by userland.
- Do not replace all array construction in Node.
- Do not expose unsafe APIs to userland JavaScript.

---

## Relationship to Other Proposals

| Proposal | Layer |
|---|---|
| Trusted Initial Intrinsics | Internal JS → JS builtins |
| Fast Internal C++ Calls | Internal JS → C++ bindings |
| Direct Array Builder (this doc) | C++ embedder code → JS arrays |

These are independent. Neither depends on the others. Each requires separate profiling and benchmarking.

---

## Existing V8 API: `v8::Array::New(elements, length)`

Before proposing new V8 API, evaluate whether the existing overload is sufficient:

```cpp
static Local<Array> v8::Array::New(
    Isolate* isolate,
    Local<Value>* elements,
    size_t length);
```

This API constructs a JS array from a pre-collected `Local<Value>[]` buffer in a single call, which may already avoid the overhead of a per-element `Set` loop.

**Any new Direct Array Builder proposal must first benchmark this existing API against the current `Set` loop for the target Node hotspot.** If `v8::Array::New(isolate, elements, length)` is sufficient, the proposal becomes a Node-side refactor rather than a new V8 API request.

### Alternatives to compare before proposing new API

1. Current `array->Set(context, index, value)` loop
2. Pre-sized `v8::Array::New(isolate, length)` + individual Set calls
3. **Existing `v8::Array::New(isolate, elements, length)`** — pre-collect elements, construct in one call
4. Node-local helper wrapping existing API
5. New V8 public API only if alternatives 1–4 are demonstrably insufficient for a measured hotspot

---

## Possible API Shapes (if existing API is insufficient)

These are tentative options for discussion. V8 reviewers should decide whether any new API belongs in public API, internal API, or not at all.

**Option A: `PushDirect` method**
```cpp
v8::Array::PushDirect(v8::Isolate* isolate, v8::Local<v8::Value> value);
```

**Option B: Builder object**
```cpp
v8::ArrayBuilder builder(isolate, expected_length);
builder.Append(value1);
builder.Append(value2);
v8::Local<v8::Array> result = builder.Finish();
```

**Option C: Pre-initialization**
```cpp
v8::Array::SetLengthAndInitializeElements(isolate, elements, length);
```

API shape is intentionally left open. Do not propose a specific public API until a real Node hotspot is measured and existing alternatives are proven insufficient.

---

## Safety Constraints

Any direct builder — whether new API or Node-local experiment — must satisfy:

- Array must be newly created by the embedder in the same call scope.
- Array must be an ordinary initial JSArray (not subclass, not proxy).
- Array must not have escaped to JavaScript before construction completes.
- Implementation must use correct GC write barriers.
- Must preserve GC heap invariants.
- Must handle allocation failures and exceptions correctly.
- Must define behavior for holes, capacity growth, and length updates.
- Must not expose partially constructed arrays to JS.

---

## Candidate Node Hotspots

Do not assert final candidates. Profile first. Possible areas:

- `fs` readdir / directory result construction
- `process` / diagnostic metadata arrays
- internal binding result arrays (multiple return values)
- module loader internal arrays
- performance / timing result arrays
- DNS lookup result arrays

**Phase 0 must profile first. Do not optimize without a measured hotspot.**

---

## MVP

1. Profile Node.js C++ code to identify a hotspot constructing large or frequent arrays via `Set` loops.
2. Benchmark the hotspot against existing `v8::Array::New(isolate, elements, length)`.
3. If existing API is sufficient: refactor Node-side code to use it — no new V8 API needed.
4. If not sufficient: prototype a Node-local helper (not public V8 API) and benchmark.
5. Only if the local experiment shows a clear, measured benefit and safety constraints are met: discuss V8 API shape with V8 reviewers.

Checklist:
- [ ] Identify C++ hotspot via profiling
- [ ] Confirm arrays are newly created and not yet observable by JS
- [ ] Benchmark existing `v8::Array::New(isolate, elements, length)` vs `Set` loop
- [ ] If existing API is insufficient: prototype Node-local helper
- [ ] Verify GC/write barriers are correct
- [ ] Add tests
- [ ] Benchmark (see plan below)

---

## Benchmark Plan

| Benchmark | Set loop | Array::New(elements) | Direct builder | Delta | Notes |
|---|---|---|---|---|---|
| small arrays (< 10 elements) | TBD | TBD | TBD | TBD | |
| medium arrays (100 elements) | TBD | TBD | TBD | TBD | |
| large arrays (1000+ elements) | TBD | TBD | TBD | TBD | |
| Node hotspot (end-to-end) | TBD | TBD | TBD | TBD | |

Include a baseline column for existing `v8::Array::New(isolate, elements, length)` before proposing anything new.

---

## Correctness Requirements

- Resulting array must be indistinguishable from a normally constructed JS Array after being returned to JavaScript.
- Length must be correct.
- Elements must be visible in correct order.
- GC must remain safe; all write barriers must be correct.
- Exceptions and allocation failures must be handled without partially observable state.
- No observable behavior changes for JS code consuming the returned array.

---

## Risks

- **Existing API may already solve the problem.** `v8::Array::New(isolate, elements, length)` may be sufficient. A new API may be unnecessary.
- **Public V8 API may be rejected** unless there is a clear measured gap versus existing alternatives.
- **GC/write-barrier mistakes are dangerous** and can cause subtle heap corruption.
- **Fast path may duplicate existing V8 internal mechanisms.** V8 may already optimize initial JSArray construction in ways that make a new API redundant.
- **Benefit may be small** if current array construction is already well-optimized.
- **API encourages unsafe embedder usage** if safety constraints are not clearly enforced.
- **Needs careful review** from V8 API, GC, and array/elements maintainers.

---

## Recommended Rollout

1. Profile Node C++ array construction hotspots.
2. Benchmark existing `v8::Array::New(isolate, elements, length)` against the current `Set` loop for the hotspot.
3. If existing API is sufficient: refactor Node-side code. No new V8 API needed.
4. If not sufficient: create a Node-local helper or experiment, not a public V8 API.
5. Benchmark the local experiment.
6. Only if benefit is large and safety is clear: discuss V8 API shape with V8 reviewers.
7. Otherwise: keep as a local optimization or drop.

**Do not propose public V8 API until a real Node hotspot and benchmark win versus `v8::Array::New(isolate, elements, length)` are demonstrated.**

---

## Key Contacts / Review Areas

Review areas (not prescriptive assignments):
- V8 API reviewers
- V8 GC / heap / write barrier reviewers — mandatory if any new direct builder API is proposed
- V8 array/elements maintainers
- Node.js subsystem owner for the selected hotspot
- Node.js performance team

Note: If the experiment uses only the existing `v8::Array::New(isolate, elements, length)`, the review surface is significantly smaller and GC review may not be required.
