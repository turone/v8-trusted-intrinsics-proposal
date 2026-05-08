# Phase 0 PoC Plan

## Goal

Build a minimal V8 fork prototype proving that trusted scripts can resolve a whitelisted builtin property load to the initial intrinsic from NativeContext.

Initial target:

- `Array.prototype.push`

Later targets:

- `Object.keys`
- `Array.isArray`
- `JSON.parse`
- `Map.prototype.get`
- `Set.prototype.add`

## Non-goals for PoC

- No full Node.js primordials removal
- No Promise/RegExp/Symbol species support
- No String primitive receiver support in first commit
- No public V8 API commitment
- No complete Maglev/TurboFan optimization in first commit

## PoC commit sequence

### Commit 1: Flag plumbing

Changes:

- Add tentative `use_initial_intrinsics` compile option
- Store bit on `SharedFunctionInfo`
- Ensure lazy functions inherit the bit
- No behavior change yet

Tests:

- Trusted script SFI has flag
- Untrusted script SFI does not have flag
- `eval` / `new Function` do not inherit flag

### Commit 2: One intrinsic — Array.prototype.push

Changes:

- Add initial intrinsic lookup table with one entry:
  - `InitialReceiverKind::kInitialJSArray + "push"`
- Patch trusted LoadIC miss path
- Return original Array.prototype.push from NativeContext / existing slot / audited source

Tests:

```js
Array.prototype.push = function hacked() {
  throw new Error('patched');
};

// trusted
const arr = [];
arr.push(1); // should not throw

// untrusted
const arr2 = [];
arr2.push(1); // should call hacked
```

### Commit 3: Static/global resolution

Add:

- `Object.keys`
- `Array.isArray`
- `JSON.parse`

Tests:

```js
Object.keys = hacked;
Object.keys({ a: 1 }); // trusted → original

const keys = Object.keys;
Object.keys = hacked;
keys({ a: 1 }); // trusted → original

globalThis.JSON = { parse: hacked };
JSON.parse("{}"); // trusted → original
```

### Commit 4: Feedback marker

Add:

- handler kind `kInitialIntrinsicLoad` in feedback slots
- feedback metadata: intrinsic id, receiver map, native context slot

Tests:

- Feedback vector shows initial intrinsic handler at trusted call sites
- Polymorphic site (initial array + subclass array) keeps correct semantics for both entries

### Commit 5: Node integration experiment

Changes:

- Build Node against patched V8
- Pass `use_initial_intrinsics = true` for selected `node:internal/*`
- Replace one Class A primordial call site, run full Node test suite

Target module candidates:

- `lib/internal/streams/writable.js`
- `lib/internal/timers.js`
- small isolated internal helper first if streams is too large

## Correctness test matrix

### Trusted vs untrusted

- trusted code ignores monkey patch on prototype
- untrusted code observes monkey patch

### Timing

- patch before first call (cold IC)
- patch after IC installed (warm IC, already recorded intrinsic handler)
- patch after optimization if Maglev/TurboFan is involved

### Tiers

- default (Ignition → Sparkplug → Maglev → TurboFan)
- `--jitless` (Ignition only — most important correctness check)
- `--no-maglev` if applicable
- stress deopt via `--stress-deopt` if available

### Receiver behavior

- ordinary initial array → initial intrinsic
- subclass array (different map) → normal lookup, subclass method called
- proxy wrapping array → normal proxy semantics
- user object with `push` property → normal lookup

### Global/static

- global replacement: `globalThis.Object = hacked` → trusted `Object.keys` unaffected
- static method replacement: `Object.keys = hacked` → trusted `Object.keys` unaffected
- extracted static method: `const k = Object.keys; Object.keys = hacked; k({})` → initial
- lexical shadowing: `function f(Object) { Object.keys({}) }` → uses parameter, not initial

### Dynamic code

- `eval("Array.prototype.push = h; [].push(1)")` inside trusted module → normal semantics
- `new Function("return [].push(1)")` → not trusted
- `vm.compileFunction` with trusted source → flag must NOT apply

## Go/no-go criteria

### Go if all of:

- trusted code consistently calls original intrinsic across all test cases
- untrusted code behavior unchanged in every variant
- works correctly under `--jitless` (no tiering dependence)
- no measurable untrusted LoadIC regression in microbenchmarks
- snapshot/native-context size delta acceptable (document the delta)
- microbenchmark shows plausible win over primordials (targeting ≤1.1× vs unprotected baseline; primordials currently run at ~2× per [poc-node-isolate-internals](https://github.com/tshemsedinov/poc-node-isolate-internals))

### No-go if any of:

- behavior depends on tiering state (different result before vs after optimization)
- polymorphic IC site disables trusted semantics for initial-map entries
- untrusted code slows down measurably
- implementation requires an unconditional SFI-bit check on every LoadIC fast path