# Benchmark Results

> **Status:** Planned. Results will be filled in once the PoC (Phase 0) is complete.
>
> **Existing reference data:** [tshemsedinov/poc-node-isolate-internals](https://github.com/tshemsedinov/poc-node-isolate-internals) provides preliminary measurements showing the `04-primordials` approach carries ~2× overhead vs. an unprotected direct-call baseline (`01-baseline`). Initial intrinsic resolution targets closing this gap while preserving protection.

## Configurations

Four builds are compared throughout all benchmarks:

| Label | Description |
|---|---|
| **A. Baseline (primordials)** | Current Node.js with primordials. Reference for correctness and performance. |
| **B. Plain JS (unprotected)** | Primordials manually replaced with plain JS, no V8 protection. Upper-bound performance. |
| **C. Trusted initial intrinsics** | Plain JS + V8 trusted initial intrinsic resolution (this proposal). |
| **D. Patched V8, feature disabled** | Same V8 patch but `use_initial_intrinsics = false`. Measures overhead of the patch itself on untrusted code. |

## Methodology

All benchmarks run on:

- **CPU:** TBD (e.g. Intel Core i7-12700K, isolated cores)
- **OS:** Ubuntu 22.04 LTS
- **V8 build:** `x64.release`, PGO off
- **Node.js:** built against patched V8

Each benchmark is run **10 times**, median reported, first run discarded (warm-up).

## Micro-benchmark: `array-push` hot loop

```js
// bench-array-push.js
const N = 1_000_000;
const arr = [];
console.time('push');
for (let i = 0; i < N; i++) arr.push(i);
console.timeEnd('push');
```

| Variant | Median (ms) | Notes |
|---|---|---|
| A. Baseline (primordials) | TBD | `ArrayPrototypePush` indirect call |
| B. Plain JS (unprotected) | TBD | Upper bound; susceptible to patching |
| C. Trusted intrinsics | TBD | Direct NativeContext slot load |
| D. Patched V8, feature off | TBD | Overhead check for non-trusted code |

## Micro-benchmark: `Object.keys` on small object

```js
const obj = { a: 1, b: 2, c: 3 };
const N = 1_000_000;
console.time('keys');
for (let i = 0; i < N; i++) Object.keys(obj);
console.timeEnd('keys');
```

| Variant | Median (ms) | Notes |
|---|---|---|
| A. Baseline (primordials) | TBD | |
| B. Plain JS (unprotected) | TBD | |
| C. Trusted intrinsics | TBD | |
| D. Patched V8, feature off | TBD | |

## Micro-benchmark: `Map.get` / `Map.set`

```js
const m = new Map();
for (let i = 0; i < 1000; i++) m.set(i, i);
const N = 1_000_000;
console.time('map-get');
for (let i = 0; i < N; i++) m.get(i % 1000);
console.timeEnd('map-get');
```

| Variant | Median (ms) | Notes |
|---|---|---|
| A. Baseline (primordials) | TBD | `MapPrototypeGet` indirect call |
| B. Plain JS (unprotected) | TBD | |
| C. Trusted intrinsics | TBD | |
| D. Patched V8, feature off | TBD | |

## Micro-benchmark: `JSON.parse`

```js
const json = '{"a":1,"b":2,"c":3}';
const N = 100_000;
console.time('json-parse');
for (let i = 0; i < N; i++) JSON.parse(json);
console.timeEnd('json-parse');
```

| Variant | Median (ms) | Notes |
|---|---|---|
| A. Baseline (primordials) | TBD | `JSONParse` indirect call |
| B. Plain JS (unprotected) | TBD | |
| C. Trusted intrinsics | TBD | |
| D. Patched V8, feature off | TBD | |

## Macro-benchmark: Node.js startup time

```bash
time node -e "1"
```

| Variant | Median startup (ms) |
|---|---|
| A. Baseline (primordials) | TBD |
| C. Trusted intrinsics | TBD |

## Macro-benchmark: Node.js internal hot paths

```bash
node benchmark/streams/
node benchmark/buffers/
node benchmark/fs/
node benchmark/timers/
```

| Benchmark | A. Baseline | C. Trusted intrinsics | Delta |
|---|---|---|---|
| streams | TBD | TBD | TBD |
| buffers | TBD | TBD | TBD |
| fs | TBD | TBD | TBD |
| timers | TBD | TBD | TBD |

## V8-level metrics

Collect via `--trace-ic`, `--print-opt-code`, or V8 counters if available:

| Metric | A. Baseline | C. Trusted intrinsics | D. Patched off |
|---|---|---|---|
| LoadIC monomorphic count | TBD | TBD | TBD |
| LoadIC polymorphic count | TBD | TBD | TBD |
| LoadIC megamorphic count | TBD | TBD | TBD |
| Deopt count | TBD | TBD | TBD |
| Optimized code size (bytes) | TBD | TBD | TBD |
| Snapshot size delta (bytes) | — | TBD | TBD |
| NativeContext size delta (bytes) | — | TBD | TBD |

## No-regression checks

These must pass before claiming the patch is safe:

| Check | Result |
|---|---|
| Untrusted LoadIC microbenchmark (D vs A) | TBD — must show ≤1% difference |
| Snapshot size delta | TBD — must be acceptable |
| NativeContext size delta | TBD — must be acceptable |
| `--jitless` correctness | TBD — trusted code must still resolve initial intrinsic |
| Monkey-patch correctness (trusted) | TBD — trusted code must ignore patch |
| Monkey-patch correctness (untrusted) | TBD — untrusted code must observe patch |

## Correctness benchmark with monkey patching

```js
// correctness-check.js
Array.prototype.push = function hacked() {
  throw new Error("prototype patched");
};

// trusted internal benchmark should still complete without throwing
// (run this script as part of a trusted module in the patched Node build)
const arr = [];
for (let i = 0; i < 1_000_000; i++) arr.push(i);
console.log("OK:", arr.length);
```
