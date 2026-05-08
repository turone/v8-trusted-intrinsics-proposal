# Security Considerations

## Threat Model

This proposal assumes a **two-party trust boundary**:

| Party | Trust level |
|---|---|
| V8 embedder (e.g. Node.js process itself) | Fully trusted |
| User land JavaScript | Untrusted |

The goal is to protect embedder-internal scripts from user-land interference, **not** to sandbox user code from each other (that is a separate concern addressed by realms / compartments).

## Attack Surfaces

### 1. Whitelist Bypass via Prototype Mutation

**Risk:** If the whitelist is checked by property name only, an attacker who can set `Array.prototype[Symbol.toPrimitive]` or similar might influence lookup semantics.

**Mitigation:** The lookup must be anchored to a specific NativeContext slot index, not to a runtime name resolution. The mapping from name → slot is done at parse/compile time and is not re-evaluated at runtime.

### 2. Reflection APIs (`Reflect.get`, `Proxy`)

**Risk:** User code wrapping a trusted script in a `Proxy` and intercepting `get` traps.

**Mitigation:** Trusted-intrinsic loads bypass the normal `[[Get]]` MOP operation entirely; they emit a direct load from NativeContext memory. Proxy traps on the receiver object are not invoked.

### 3. Script Tag Forgery

**Risk:** If user code can call `ScriptCompiler::Compile` with the `kTrustedIntrinsics` flag, it can obtain initial intrinsics for its own scripts.

**Mitigation:** The `kTrustedIntrinsics` flag is a C++ API option. It is never surfaced to JavaScript. Only the embedder controls which scripts are marked trusted (e.g. Node.js marks files in `lib/internal/` at build time).

### 4. Deserialization / Snapshot Tampering

**Risk:** If the V8 snapshot is replaced or tampered with, the NativeContext slots may contain attacker-controlled values.

**Mitigation:** Snapshot integrity is already a concern for V8 embedders. This proposal does not weaken existing snapshot security; it relies on the same trust assumptions already required to run V8 at all.

### 5. Cross-Realm Confusion

**Risk:** Scripts that traverse realm boundaries (e.g. via `iframe`, `vm.createContext`) may observe unexpected intrinsic values.

**Mitigation:** Trusted-intrinsic resolution is scoped to the NativeContext of the script's own realm. Cross-realm property loads follow normal semantics.

## Security Review Checklist

- [ ] No JavaScript-observable API to enable trusted mode.
- [ ] Whitelist is hard-coded in C++ and cannot be extended from JS.
- [ ] Trusted flag does not survive `eval`, `Function()`, or dynamic code generation.
- [ ] Fuzz the boundary: run V8's existing `jsfunfuzz` / `v8-foozzie` with trusted-intrinsics enabled.
- [ ] Review interaction with `--harmony-*` flags and experimental features.

---

## Trust Boundary

### Trusted (may compile with `use_initial_intrinsics = true`)

- Node.js bootstrap modules
- `node:internal/*` built-in modules
- selected built-in internal helpers (e.g. `node:_http_common`)

### Untrusted (always `use_initial_intrinsics = false`)

- user application code
- npm packages
- `eval` — even when called from a trusted module
- `new Function` — same
- `vm.Script` / `vm.compileFunction`
- custom ESM loaders / `--loader`
- `--require` hooks
- worker threads (unless explicitly re-trusted by embedder)

---

## Invariants

### Invariant 1: Userland code never gets the flag

Every userland compilation path must unconditionally set `use_initial_intrinsics = false`.

Audit paths:
- CommonJS user modules (CJS loader)
- ESM user modules (ESM loader)
- `vm.Script` / `vm.compileFunction`
- `eval` / `new Function`
- custom ESM loaders
- `--require` / `--loader` flags
- `--expose-internals` (must not allow userland to re-use trusted scripts)
- worker threads

### Invariant 2: Dynamic code is never trusted

Even inside a trusted `node:internal/*` module:

```js
// trusted internal module
eval("Array.prototype.push = hacked; [].push(1)");
// evaluated code must use normal JS semantics
```

The parser/compiler sets `use_initial_intrinsics = false` unconditionally for all `eval`, `new Function`, and `vm.compileFunction` paths. This is a hard policy, not a runtime check.

### Invariant 3: Only whitelisted named loads change

Affected (named property loads on proven initial builtin receivers/constructors):
```js
arr.push      // initial JSArray receiver + whitelist entry
Object.keys   // initial Object constructor + whitelist entry
JSON.parse    // initial JSON namespace + whitelist entry
```

Not affected:
```js
obj.foo                // unknown receiver map → normal lookup
obj[dynamicName]       // dynamic key → always normal
obj[Symbol.iterator]   // symbol key → always normal
for (const k in obj)   // enumeration → always normal
Reflect.ownKeys(obj)   // reflection → always normal
```

### Invariant 4: Non-initial receivers use normal lookup

```js
class A extends Array {
  push() { return "custom"; }
}
const a = new A();
a.push(1); // subclass map ≠ initial JSArray map → normal subclass lookup
```

The map check is at the IC level. Subclasses, proxies, and user-defined objects always fall through to normal prototype chain lookup.

### Invariant 5: Argument semantics are unchanged

Trusted initial intrinsics only change **which function is called**. They do not suppress:
- proxy traps inside builtin execution
- getters or setters on arguments
- user-provided callbacks
- JSON revivers
- descriptor semantics or inherited descriptor properties
- argument coercions (`ToInteger`, `ToString`, etc.)

---

## Attack / Bypass Scenarios

### Scenario: User mutates prototype

```js
Array.prototype.push = hacked;
```

Expected: trusted code with initial JSArray receiver calls the NativeContext-original push; untrusted code calls `hacked`.

### Scenario: User replaces global

```js
globalThis.Object = hacked;
```

Expected: trusted `Object.keys({})` resolves the global load to the initial Object constructor from NativeContext, unaffected.

### Scenario: User shadows with lexical binding

```js
function f(Object) {
  return Object.keys({});  // uses the parameter Object, NOT initial Object
}
```

Expected: lexical/parameter bindings are always resolved normally. The initial-intrinsic global path activates only for actual global property loads (`LoadGlobalIC`), not for statically-resolved lexical names.

### Scenario: User passes subclass instance to trusted code

```js
class MyArray extends Array {}
const a = new MyArray();
internalModule.process(a);  // trusted code calls a.push(x)
```

Expected: `a` has the subclass map (not an initial JSArray map), so IC falls through to normal prototype lookup and calls the subclass `push` if defined, or `Array.prototype.push` via normal chain walk. This is intentional — the map check distinguishes initial from subclass.

### Scenario: User passes proxy

```js
const p = new Proxy([], handler);
internalModule.process(p);
```

Expected: proxy is not an initial JSArray instance (different receiver shape), so IC falls through to normal proxy semantics.

### Scenario: Extracted static method

```js
const keys = Object.keys;
Object.keys = hacked;
keys({ a: 1 });  // trusted → returns initial Object.keys value (from NativeContext)
```

Expected: the property load `Object.keys` in trusted code resolved to the initial function from NativeContext. The local `keys` variable holds the initial function. Subsequent mutation of `Object.keys` does not affect the already-resolved value.

---

## Node Migration Security Checklist

For each primordial call site being replaced with plain JS, verify:

- [ ] **Receiver origin** — is the receiver created internally (Class A) or user-provided (Class B/C)?
- [ ] **Subclass/proxy possibility** — can the receiver be a subclass, proxy, or array-like (Arguments, NodeList)?
- [ ] **Symbol hooks** — does the operation dispatch through `Symbol.species`, `Symbol.iterator`, `Symbol.match`, `Symbol.split`, `Symbol.replace`, or `Symbol.toPrimitive`?
- [ ] **Callbacks** — does the operation accept and invoke a user-provided callback (e.g., `Array.prototype.map` with a mapper)?
- [ ] **Descriptor semantics** — does the operation read or write property descriptors where inherited descriptor properties (`value`, `get`, `set`) from `Object.prototype` could interfere?
- [ ] **Semantic equivalence** — does plain-JS behavior match primordial behavior for all reachable inputs (especially subclass + proxy)?

---

## Risk Classification

### Low risk — safe to rewrite

Class A: receiver is created internally and never escapes to userland before use.

```js
const queue = [];
queue.push(item);  // internal array, initial map guaranteed
```

### Medium risk — semantic review required

Class B: receiver is user-provided but expected to be a plain builtin instance. Normal JS semantics differ from primordial semantics for subclass receivers.

```js
function processItems(items) {
  items.push(newItem);  // works if items is a plain Array; subclass gets normal JS semantics
}
```

Each Class B site requires: explicit review, tests for subclass/proxy edge cases, and confirmation that Node.js wants normal-JS semantics there.

### High risk — keep primordials

Class C: receiver may be array-like, subclass, proxy, or the operation involves Symbol dispatch.

```js
ArrayPrototypeSlice(arrayLike);     // array-like; IC map check will fail
RegExpPrototypeExec(regexp, str);   // Symbol.match observable on subclass
PromiseResolve(x);                  // thenable assimilation; Symbol.species
```

---

## Required Tests

- trusted vs untrusted (same mutation, different behavior)
- `eval` and `new Function` inside trusted module use normal semantics
- `vm.Script` and `vm.compileFunction` always untrusted
- worker threads: flag not inherited by worker scripts
- `--expose-internals`: does not grant trusted mode to userland
- subclass fallback (subclass map → normal lookup)
- proxy fallback (proxy → normal proxy semantics)
- lexical shadowing (parameter/const shadows global → uses binding, not initial)
- extracted static method (initial function captured in variable before mutation)
- polymorphic IC site (initial + non-initial receiver at same call site → both correct)
