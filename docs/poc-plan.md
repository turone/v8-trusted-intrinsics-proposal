# PoC Plan — Trusted Initial Intrinsics

## Goal

Demonstrate, on a custom V8 build, that a trusted script can call `Array.prototype.push` and get the initial built-in implementation even after user code replaces it, **without** using primordials.

## Phases

### Phase 1 — V8 Build Setup

- [ ] Fork or patch V8 at a stable release tag (e.g. `12.x`).
- [ ] Add a `ScriptCompiler::CompileOptions` flag: `kTrustedIntrinsics`.
- [ ] Confirm clean build with `tools/dev/gm.py x64.release`.

### Phase 2 — NativeContext Slot Lookup

- [ ] Identify the four MVP slots in `src/objects/contexts.h`:
  - `ARRAY_PUSH_INDEX`
  - `OBJECT_KEYS_INDEX`
  - `ARRAY_IS_ARRAY_INDEX`
  - `JSON_PARSE_INDEX`
- [ ] Add a helper `NativeContext::InitialIntrinsic(name)` that maps a property-name string to its slot and returns the initial value.

### Phase 3 — Bytecode / IC Patch

- [ ] In `src/compiler/bytecode-graph-builder.cc` (or the relevant IC path), detect when the current script has the `kTrustedIntrinsics` flag and the load target is a whitelisted name.
- [ ] Emit a direct load from the NativeContext slot instead of a normal property load.

### Phase 4 — Node.js Integration Shim

- [ ] Write a small Node.js add-on (`binding.cc`) that:
  1. Compiles a JS string with `kTrustedIntrinsics`.
  2. Runs user code that replaces `Array.prototype.push`.
  3. Calls the trusted script and asserts it still uses the original `push`.

### Phase 5 — Validation

- [ ] Write a test in `test/unittests/` that exercises the path.
- [ ] Run `tools/run-tests.py --outdir=out/x64.release unittests`.
- [ ] Collect flamegraph before/after to confirm no perf regression.

## Success Criteria

1. Trusted script returns the original `Array.prototype.push` descriptor after user replacement.
2. No observable change in behavior for non-trusted scripts.
3. No perf regression (≥ 0% change) on Speedometer or `perf/array-push` micro-benchmark.

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| IC complexity — trusted flag may not survive optimization tiers | Add a deopt guard that re-checks the flag on OSR |
| Snapshot compatibility — NativeContext slots differ between snapshots | Pin to a specific `--startup-data-hash` in tests |
| API surface — embedder ABI break | Gate behind a compile-time `V8_TRUSTED_INTRINSICS` macro |
