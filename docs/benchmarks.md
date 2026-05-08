# Benchmark Results

> **Status:** Planned. Results will be filled in once the PoC (Phase 5) is complete.

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
| Baseline (primordials) | TBD | `ArrayPrototypePush` indirect call |
| Trusted intrinsics | TBD | Direct NativeContext slot load |
| Plain JS (no protection) | TBD | Reference; susceptible to patching |

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
| Baseline (primordials) | TBD | |
| Trusted intrinsics | TBD | |
| Plain JS | TBD | |

## Macro-benchmark: Node.js startup time

Measure time-to-first-byte for a minimal HTTP server that imports several internal modules normally guarded by primordials.

| Variant | Median startup (ms) |
|---|---|
| Baseline (primordials) | TBD |
| Trusted intrinsics | TBD |

## Expected Outcome

We expect trusted intrinsics to match or outperform primordials because:

1. The property load is resolved at compile time to a known NativeContext slot.
2. The JIT can inline the call target without an indirect indirection layer.
3. No snapshot of the entire prototype chain is needed at startup.
