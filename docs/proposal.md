# Proposal: Trusted Initial Intrinsics for V8 Embedders

## Problem

Node.js internal modules (`lib/internal/*`) use **primordials** — ~600 saved references to original built-in functions — to protect against userland monkey patching.

```js
// Current: safe but unreadable
const { ArrayPrototypePush, MapPrototypeGet, StringPrototypeSlice } = primordials;

function enqueue(item) {
  ArrayPrototypePush(queue, item);
}
```

Two consequences:

1. **Performance** — `uncurryThis` wraps every call in `Function.prototype.call`. TurboFan cannot inline through indirect calls. Inline caches become megamorphic because hundreds of different builtins flow through one `call` wrapper. Preliminary measurements ([tshemsedinov/poc-node-isolate-internals](https://github.com/tshemsedinov/poc-node-isolate-internals)) show the `04-primordials` variant carries ~2× overhead vs. the unprotected direct-call baseline (`01-baseline`), confirming the real-world cost.

2. **Developer experience** — code is unreadable, hard to review, hard to contribute. `ArrayPrototypePush(arr, x)` instead of `arr.push(x)` across thousands of lines in Node core.

## Goal

Allow Node.js internal modules to write **plain JavaScript** for a large, explicitly whitelisted subset of builtin operations, while getting primordial-like resolution to initial intrinsic values. V8 resolves known operations on known builtin receivers against their **initial values from the native context**, regardless of userland mutations.

```js
// After: plain JavaScript, primordial-like protection for whitelisted operations
function enqueue(item) {
  queue.push(item);  // V8 always calls the original Array.prototype.push
}
```

This proposal does not make trusted JavaScript "immune to userland objects." It only changes resolution of whitelisted intrinsic property loads when the IC can prove the base is an initial builtin receiver/constructor. All argument processing, proxy traps, callbacks, getters, and non-whitelisted property accesses retain normal JavaScript semantics.

This is **not** a transparent drop-in replacement for every primordial call. It covers only operations where V8 can prove the receiver/global is an initial built-in object or instance. Existing primordials or Safe* helpers remain necessary for:
- Operations on user-provided array-likes, subclasses, or proxies
- Spec methods that perform additional user-observable lookups (RegExp symbol hooks, Promise thenable assimilation, Symbol.species)

### Scope summary

This proposal changes only named property loads in trusted scripts when **all** of the following are true:

1. The script is compiled with `use_initial_intrinsics = true`.
2. The base is an initial builtin receiver, constructor, or namespace object (verified by map/identity check).
3. The property name is in the whitelist (intrinsics table).
4. The IC records an initial-intrinsic handler in feedback.
5. Dynamic code (`eval`, `new Function`) is excluded — always untrusted.

It does **not** change argument semantics, callbacks, proxy traps, lexical bindings, subclass behavior, or arbitrary property access.

## Non-goals

- Do not change JavaScript semantics for userland code
- Do not skip prototype dependencies for arbitrary property lookups
- Do not optimize `obj.foo()` when `obj` comes from userland
- Do not cover user-defined prototypes or subclasses
- Do not create a new security boundary — reuse existing `node:internal/*` boundary
- Do not aim to eliminate primordials entirely in MVP — Class C call sites and complex spec operations will keep primordials/Safe* helpers

## Why "Skip Prototype Dependencies" Is Not Enough

The previous version of this proposal suggested skipping `CompilationDependencies::DependOnStablePrototypeChains` for internal scripts. This has a fundamental flaw:

### Tier inconsistency

V8 has a multi-tier pipeline:

```
Ignition (interpreter) → Sparkplug (baseline) → Maglev (mid-tier) → TurboFan (top-tier)
```

Skipping dependencies only affects Maglev and TurboFan. Before optimization:

```
Ignition:   queue.push(item) → runtime prototype lookup → sees PATCHED push
Sparkplug:  queue.push(item) → runtime prototype lookup → sees PATCHED push
Maglev:     queue.push(item) → inlined original push (no deopt guard)
TurboFan:   queue.push(item) → inlined original push (no deopt guard)
```

Behavior depends on tiering state. Under `--jitless` or before the function is hot — monkey patching affects internal code. After optimization — it doesn't. This is unacceptable: protection must be deterministic, not "if the optimizer already ran."

Primordials work consistently across all tiers: `ArrayPrototypePush` is a saved reference, always calls the original, in Ignition and TurboFan alike.

### Globals and static methods not covered

`Object.keys(o)` involves two mutable lookups:

```js
Object      // global property — can be replaced
Object.keys // property on constructor — can be replaced
```

`DependOnStablePrototypeChains` does not cover either. If userland does `Object.keys = hacked`, a dependency-only approach doesn't catch this.

### Too broad

A blanket `if (is_internal) skip_dependency()` disables prototype dependency for ALL property lookups in internal code, including lookups on userland objects. If internal code calls `userObj.foo()`, it would keep a stale inline forever — incorrect behavior.

## Correct Approach: Initial Intrinsic Resolution

The protection must happen **at every tier**, not just in optimizing compilers. V8 must resolve known builtin operations against their **initial values from the native context**, regardless of what the prototype chain currently contains.

### V8 already stores many initial intrinsics

V8's `NativeContext` holds references to many initial constructors, prototypes, maps, and builtin entry points:

```cpp
// Already exists in V8 (examples):
NativeContext::initial_array_prototype()       // original Array.prototype
NativeContext::initial_object_prototype()       // original Object.prototype
NativeContext::array_function()                // Array constructor
NativeContext::object_function()               // Object constructor
// Many but not necessarily ALL JS-visible builtin functions have dedicated slots.
// Phase 0 must audit which desired operations already have stable context slots.
// Missing entries can either be excluded from MVP or added as new NativeContext slots.
// Adding slots is straightforward mechanically but affects context layout,
// initialization, snapshots, and memory per context.
// Phase 0 should prefer existing slots where possible.
// Slot names like ARRAY_PUSH_INDEX are illustrative;
// Phase 0 will use actual V8 NativeContext indices where they exist.
```

These references represent the initial context state and are not affected by ordinary userland mutations of global/prototype properties. Torque/CSA builtins already use these slots to call original builtins. The task is to make this available to trusted embedder JavaScript.

### How it works across tiers

For a script compiled with `use_initial_intrinsics = true`:

```
Tier        Behavior for queue.push(item)
─────────────────────────────────────────────────────────────
Ignition    IC miss → check receiver map → is initial JSArray map?
            YES → load push from NativeContext::array_push() (initial)
            NO  → normal prototype chain lookup (userland object)

Sparkplug   Same IC-based property access machinery as Ignition;
            once IC feedback/handler is initial-intrinsic handler,
            Sparkplug-emitted code observes the same behavior

Maglev      Recognizes initial builtin call → inlines
            No prototype dependency registered
            Map check remains (type safety)

TurboFan    Same as Maglev — permanent inline, no prototype dependency
```

Key difference from the previous approach: **Ignition and Sparkplug also resolve to the initial intrinsic**. The behavior is consistent across all tiers.

### When it activates and when it doesn't

The intrinsic resolution triggers **only** when:

1. The receiver has one of the **initial builtin instance maps** for its type (e.g., one of the native-context initial JSArray maps for ordinary arrays across supported elements kinds — packed smi, packed double, packed object, holey variants, etc.). Subclass arrays have different maps and are excluded
2. The property being accessed is a **known intrinsic** stored in the native context
3. The script is compiled with `use_initial_intrinsics = true`

```js
// Internal module (use_initial_intrinsics = true):

const queue = [];
const cache = new Map();

queue.push(item);        // ✅ initial JSArray map + known intrinsic → original push
cache.get(key);           // ✅ initial Map map + known intrinsic → original get
str.slice(0, 64);         // ✅ initial String map + known intrinsic → original slice
Object.keys(o);           // ✅ Object is initial Object constructor → original keys

userObj.foo();            // ❌ not a known builtin map → normal lookup, prototype dependency applies
subclassArr.push(item);  // ❌ subclass map, not initial JSArray → normal lookup
proxyArr.push(item);     // ❌ proxy, not initial JSArray → normal lookup
```

## V8 Patch — What to Change

### 1. ScriptOriginOptions — add the flag (~5 lines)

```cpp
// include/v8-script.h
// Use ScriptOriginOptions (existing bit field) instead of a new bool:
class V8_EXPORT ScriptOriginOptions {
 public:
  // Existing bits:
  bool IsSharedCrossOrigin() const;
  bool IsOpaque() const;
  bool IsWasm() const;
  bool IsModule() const;
  // New:
  bool UseInitialIntrinsics() const;  // ← add this
};
```

Using `ScriptOriginOptions` (existing bit field) avoids constructor signature changes.

**Note:** The API shape is tentative. `ScriptOriginOptions` is shown for concreteness. The final placement should be decided with V8 reviewers. Alternatives include: `ScriptCompiler::CompileOptions` bit, embedder-private compile option, experimental flag-gated API, or a Node.js-only downstream experiment before upstreaming.

### 2. SharedFunctionInfo — store the flag (~5 lines)

```cpp
// src/objects/shared-function-info.h
DECL_BOOLEAN_ACCESSORS(use_initial_intrinsics)
// 1 bit in flags
```

Propagated to all functions parsed from this script, including lazy-parsed inner functions.

The flag is a **semantic compile option**. Any cache that reuses bytecode, feedback metadata, optimized code, or deserialized SFIs must distinguish trusted and untrusted compilations (see Compilation Cache section).

### 3. Inline Cache handlers — resolve to initial intrinsic (~40–60 lines)

This is the **core change**. It makes the behavior consistent across all tiers.

```cpp
// src/ic/ic.cc or src/ic/accessor-assembler.cc
// In the LoadIC / KeyedLoadIC handler:

// When handling a property load for a trusted script:
if (shared_function_info->use_initial_intrinsics()) {
  // Check: is the receiver map an initial builtin instance map?
  if (IsInitialBuiltinInstanceMap(receiver_map)) {
    // Check: does this property name have a known intrinsic slot?
    int intrinsic_index = LookupInitialIntrinsicSlot(
        receiver_map, property_name);
    if (intrinsic_index != kNoIntrinsic) {
      // Load from native context initial intrinsic slot
      // instead of walking the prototype chain
      return native_context->get(intrinsic_index);
    }
  }
  // Fall through to normal IC for non-builtin receivers
}
```

The IC handler is **pinned**: it does not update when the prototype changes. For non-builtin receivers, normal IC behavior applies (including prototype dependencies and deopt).

### 4. Intrinsic lookup table (~30 lines)

A table mapping (builtin instance map, property name) → native context slot:

```cpp
// src/objects/initial-intrinsics-table.h (new file)

// The actual check is map/native-context based, not merely InstanceType based.
// InstanceType alone is insufficient because subclasses can share JS_ARRAY_TYPE.

enum class InitialReceiverKind {
  kInitialJSArray,
  kInitialJSMap,
  kInitialJSSet,
  // ...
};

struct InitialIntrinsicEntry {
  InitialReceiverKind receiver_kind;  // map-based, not InstanceType
  const char* property_name;
  int native_context_slot;
};

static constexpr InitialIntrinsicEntry kInitialIntrinsics[] = {
  {InitialReceiverKind::kInitialJSArray, "push",  Context::ARRAY_PUSH_INDEX},
  {InitialReceiverKind::kInitialJSArray, "pop",   Context::ARRAY_POP_INDEX},
  {InitialReceiverKind::kInitialJSArray, "shift", Context::ARRAY_SHIFT_INDEX},
  {InitialReceiverKind::kInitialJSMap,   "get",   Context::MAP_GET_INDEX},
  {InitialReceiverKind::kInitialJSMap,   "set",   Context::MAP_SET_INDEX},
  {InitialReceiverKind::kInitialJSMap,   "has",   Context::MAP_HAS_INDEX},
  {InitialReceiverKind::kInitialJSSet,   "add",   Context::SET_ADD_INDEX},
  {InitialReceiverKind::kInitialJSSet,   "has",   Context::SET_HAS_INDEX},
  // ... ~15 MVP v0 entries
};

// Receiver check uses native-context initial maps, not InstanceType:
bool IsInitialBuiltinInstanceMap(NativeContext ctx, Map receiver_map,
                                 InitialReceiverKind kind) {
  switch (kind) {
    case InitialReceiverKind::kInitialJSArray:
      // Checks against ALL initial array maps (packed smi, packed double,
      // packed object, holey variants). Subclass arrays have different maps.
      return ctx->IsInitialArrayMap(receiver_map);
    case InitialReceiverKind::kInitialJSMap:
      return receiver_map == ctx->initial_js_map_map();
    // ...
  }
}
```

This is a **whitelist**. Only known builtins on known receiver maps (verified via native context, not InstanceType) are resolved via initial intrinsics. Everything else → normal prototype lookup.

### 5. Initial global and static property resolution (~30 lines)

For expressions like `Object.keys(o)`, there are **two** mutable lookups:

```js
Object      // LoadGlobal — global property, can be replaced
Object.keys // LoadNamed on the constructor — own property, can be mutated
```

Resolving `Object` to the initial Object constructor alone is insufficient because `Object.keys` is a mutable own property on that constructor. The mode must resolve **both** steps:

1. **LoadGlobalIC** — for known global names (`Object`, `Array`, `JSON`, `Reflect`, `Promise`, `Map`, `Set`, `Symbol`), return the initial constructor/namespace from native context.

**Note:** Only actual global property loads are affected. Lexical bindings or local aliases shadowing global names retain normal lexical semantics:

```js
// Parameter shadows global — NOT resolved to initial Object:
function f(Object) {
  return Object.keys({});  // uses the parameter, not initial Object
}

// const/let shadows global — NOT resolved:
const JSON = { parse: custom };
JSON.parse(text);  // uses the local binding
```

```cpp
// In LoadGlobalIC for trusted scripts:
if (shared_function_info->use_initial_intrinsics()) {
  int slot = LookupInitialGlobalSlot(property_name);
  if (slot != kNoIntrinsic) {
    return native_context->get(slot);  // initial Object constructor
  }
}
```

2. **LoadIC on initial constructor/namespace** — for known static properties on these objects, return the initial function slot:

```cpp
// In LoadIC for trusted scripts, when receiver is initial constructor/namespace:
if (shared_function_info->use_initial_intrinsics() &&
    IsInitialConstructorOrNamespace(receiver)) {
  int slot = LookupInitialStaticIntrinsic(receiver, property_name);
  if (slot != kNoIntrinsic) {
    return native_context->get(slot);  // initial Object.keys function
  }
}
```

The intrinsics table must have entries for both instance methods AND static methods:

```cpp
// Instance methods: {receiver_kind, property_name, slot}
{InitialReceiverKind::kInitialJSArray, "push", Context::ARRAY_PUSH_INDEX}

// Static methods: {initial_constructor_context_slot, property_name, intrinsic_slot}
struct InitialStaticIntrinsicEntry {
  int receiver_context_slot;   // e.g., Context::OBJECT_FUNCTION_INDEX
  const char* property_name;
  int intrinsic_context_slot;  // e.g., Context::OBJECT_KEYS_INDEX
};

{Context::OBJECT_FUNCTION_INDEX, "keys",    Context::OBJECT_KEYS_INDEX}
{Context::JSON_OBJECT_INDEX,    "parse",    Context::JSON_PARSE_INDEX}
{Context::ARRAY_FUNCTION_INDEX, "isArray",  Context::ARRAY_IS_ARRAY_INDEX}

// Static lookup check:
// if (receiver == native_context->get(entry.receiver_context_slot))
//   return native_context->get(entry.intrinsic_context_slot);
```

### 6. Maglev/TurboFan — inline without prototype dependency (~20 lines)

When the optimizing compilers see a call site that the IC already resolved as an initial intrinsic:

```cpp
// In CompilationDependencies or JSCallReducer:
if (shared_info->use_initial_intrinsics() &&
    IsInitialIntrinsicCallSite(feedback)) {
  // Inline the builtin directly from native context
  // No prototype stability dependency needed
  // Map check on receiver still required
}
```

This is the same optimization from the previous proposal, but now it is **only applied to call sites already identified as initial intrinsics by the IC**, not to all property lookups.

### Direct calls

For direct calls like `arr.push(x)`, the trusted LoadIC resolves the property load to the initial intrinsic function. The call feedback then observes a known builtin target with the original receiver. Optimizing compilers may inline only when both the load feedback and call feedback agree on the initial-intrinsic target and receiver map guard.

This distinction matters: the proposal lives in **LoadIC** (property resolution), not in a separate CallIC override. The call side consumes whatever function the load produced — which, for trusted initial-intrinsic loads, is always the native-context original.

### 7. Feedback representation

Trusted intrinsic resolution must be visible in IC feedback as a dedicated handler kind or metadata:

```cpp
// When the trusted LoadIC resolves a property load to an initial intrinsic,
// the feedback slot records:
//   - handler kind: kInitialIntrinsicLoad
//   - intrinsic id / native context slot
//   - receiver map (for the map guard)
//
// Optimizing compilers consume this feedback and can inline the
// corresponding builtin without prototype-chain dependencies.
```

**Polymorphic call sites:** If a site becomes polymorphic with both initial-intrinsic receivers and normal receivers:

```js
function f(x) {
  x.push(1);
}
f([]);              // initial array → intrinsic handler
f(subclassArr);     // subclass → normal handler
```

Compiler options:
1. Generate guarded branches: intrinsic for initial-map cases, normal call for others.
2. Decline the intrinsic specialization for that site and use a generic handler.

**Correctness invariant:** Initial builtin maps must always resolve to initial intrinsics, even at polymorphic sites. Non-initial maps use normal handlers. For PoC, option 2 (decline at polymorphic) means declining **compiler-level intrinsic inlining** at polymorphic sites, not disabling IC-level initial-intrinsic resolution. The semantic invariant remains: initial builtin maps still resolve to initial intrinsics at the IC level; only the optimization (inlining) is skipped. Production should support option 1.

### 8. Snapshot serialization (~10 lines)

```cpp
// The flag is preserved in the startup snapshot.
// Internal modules compiled at build time retain use_initial_intrinsics.
```

### Total patch size

**PoC (Phase 0):** ~300–700 lines (depending on whether feedback metadata and cache compatibility are included)
```
Component                                Lines    What
────────────────────────────────────────────────────────────────────
include/v8-script.h                      ~5       ScriptOriginOptions bit
src/objects/shared-function-info.h       ~5       SFI bit field
src/ic/ic.cc (or accessor-assembler)     ~60-80   IC handler for initial intrinsics
src/objects/initial-intrinsics-table.h   ~50      Whitelist table (MVP subset)
src/ic/ic.cc (global IC + static IC)     ~40-50   Global + static property resolution
src/snapshot/                            ~10      Snapshot preservation
────────────────────────────────────────────────────────────────────
Core logic                               ~170-200
Tests (cctest + mjsunit)                 ~150-300
Documentation / design doc               ~100
────────────────────────────────────────────────────────────────────
PoC total                                ~300-700
```

**Upstream CL (Phase 2, production quality):** ~800–1500+ lines
```
Component                                Lines    What
────────────────────────────────────────────────────────────────────
All PoC components above                 ~170-200 Core logic
src/compiler/compilation-dependencies.cc ~15      Skip dependency for intrinsic sites
src/maglev/maglev-graph-builder.cc       ~20-30   Maglev intrinsic inline
src/compiler/js-call-reducer.cc          ~20-30   TurboFan reducer for intrinsic calls
Edge cases / defensive checks            ~50-100  Subclass, proxy, primitive receivers
src/objects/feedback-vector*             ~20-40   Feedback handler metadata
src/codegen/ or compilation-cache        ~15-25   Cache key compatibility
────────────────────────────────────────────────────────────────────
Core logic (production)                  ~350-500
Comprehensive tests                      ~300-600 All tiers, jitless, fuzzing hooks
API documentation (v8.h comments)        ~50
────────────────────────────────────────────────────────────────────
Upstream CL total                        ~900-1700+
```

Note: Previous estimate of ~250–350 was for core logic only. Production CLs require extensive test coverage (V8 standard is ~2x test lines per feature line), documentation, and edge case handling.

## Coverage

### MVP — Phase 0/PoC Scope

These are operations where receiver map check + intrinsic table lookup is straightforward: no user-observable side channels, no Symbol.species dispatch, no thenable assimilation.

#### Prototype methods (MVP v0)

```
Receiver type   Methods                                    NativeContext slot
──────────────────────────────────────────────────────────────────────────────
JSArray         push, pop, shift, unshift                  ARRAY_PUSH_INDEX...
Map             get, set, has, delete, clear               MAP_GET_INDEX...
Set             add, has, delete, clear                    SET_ADD_INDEX...
String          slice, indexOf, trim                       STRING_SLICE_INDEX...
```

#### Static methods (MVP v0)

```
Global/Static                     NativeContext slot
──────────────────────────────────────────────────────────
Array.isArray                     ARRAY_IS_ARRAY_INDEX
Object.keys                       OBJECT_KEYS_INDEX
Object.create                     OBJECT_CREATE_INDEX
Object.defineProperty             OBJECT_DEFINE_PROPERTY_INDEX
JSON.parse                        JSON_PARSE_INDEX
```

#### Why these are safe for MVP

- **Array push/pop/shift/unshift** — included because lookup protection only targets the method identity on exact ordinary initial arrays. The builtins still perform their normal element operations and observable effects (e.g., length update, element moves for shift/unshift).
- **Map/Set** — operate on internal slots, no prototype lookup on arguments.
- **String slice/indexOf/trim** — trusted resolution protects the method lookup. The builtins still perform normal ECMAScript coercions on arguments (`ToInteger`, `ToString`). Included because they do not dispatch through RegExp/Symbol.match-style hooks.
- **Array.isArray** — pure type check.
- **Object.keys/create/defineProperty** — trusted resolution protects the function identity. The builtins retain normal spec behavior including proxy traps if receiver is a proxy. **Important for migration:** `Object.defineProperty` migration does not remove the need for null-prototype descriptor objects. Trusted resolution protects only the identity of `Object.defineProperty`, not the descriptor's inherited properties (e.g., polluted `Object.prototype.value`/`get`/`set`). Similarly, `Object.create` migration does not sanitize the optional properties descriptor object; if descriptors are supplied, their normal descriptor semantics and inherited properties still apply.
- **JSON.parse** — trusted resolution protects the lookup. If a reviver function is provided, it is called normally. No Symbol.* hooks.

**Important:** "safe for MVP" means safe to include in the intrinsic whitelist. Each Node.js migration must still classify call sites by receiver/argument provenance (see Migration Classifier). Proxy receivers will trigger proxy traps via the normal builtin path — the IC only protects function identity.

#### MVP candidates after audit

The following operations are likely safe but require verifying V8 fast paths and spec edge cases:

```
Array.prototype.slice          — Symbol.species: safe only for exact initial JSArray receiver
                                 (no subclass). V8 fast path must not invoke ArraySpeciesCreate.
Array.prototype.indexOf        — Safe if no holes + no exotic elements. Audit V8 fast path.
Array.prototype.includes       — Same as indexOf.
String.prototype.includes      — Checks argument for Symbol.match (RegExp detection).
String.prototype.startsWith    — Same Symbol.match check as includes.
String.prototype.endsWith      — Same Symbol.match check as includes.
Object.freeze                  — Invokes proxy traps on proxy receiver (same as Object.keys).
JSON.stringify                 — Invokes toJSON, replacers, property getters, proxy traps.
                                 Trusted resolution protects JSON.stringify lookup only.
Reflect.apply                  — Call target and arguments retain normal JS semantics.
                                 Trusted resolution protects Reflect.apply lookup only.
```

### Future Phases — Desirable but Complex

The following operations would benefit from initial intrinsic resolution but involve additional complexity that should be addressed after the MVP is proven:

#### Phase A: Extended Array/String operations

```
Receiver type   Methods                      Complexity
──────────────────────────────────────────────────────────────────────────────
JSArray         map, filter, forEach, find,  Symbol.species on subclass,
                reduce, splice, join         ArraySpeciesCreate observable
String          split, replace               Invoke Symbol.split/Symbol.replace
                                             on RegExp argument — two-sided dispatch
```

**Why deferred:** `Array.prototype.map` on a subclass triggers `Symbol.species` to determine the returned array constructor. `String.prototype.split/replace` check if the separator has `Symbol.split/Symbol.replace` methods. These are user-observable side channels that the IC-level resolution alone cannot suppress.

#### Phase B: Promise methods

```
Method                     Complexity
──────────────────────────────────────────────────────────────────────────────
Promise.then/catch/finally Thenable assimilation: reads `.then` on resolved value.
                           Must prove the return/argument is not a user thenable.
Promise.resolve/reject     Promise.resolve checks Symbol.species + constructor.
Promise.all/allSettled/race Iterate input, read `.then` on each element.
```

**Why deferred:** Promise operations perform thenable assimilation — they look up `.then` on arbitrary values. Protecting `Promise.prototype.then` alone doesn't prevent userland objects from being treated as thenables. Requires proving that internal code only passes known Promise instances (Class A pattern — see Migration Classifier).

#### Phase C: RegExp, iterators, Symbol hooks

```
Method/Property            Complexity
──────────────────────────────────────────────────────────────────────────────
RegExp.test/exec           Symbol.match, Symbol.search observable on subclass
Symbol.iterator            Per-type protocol, generator .next() is user-patchable
Symbol.species             Affects Array/Promise/TypedArray subclass construction
Object.assign              Reads Symbol keys, invokes getters
Array.from                 Reads Symbol.iterator on argument
```

**Why deferred:** These expose user-observable dispatch through well-known Symbols. A RegExp subclass can override `Symbol.match`. An iterable can have a mutated `[Symbol.iterator]`. Protecting these requires either proving the receiver is not a subclass (map check covers this for most cases) or suppressing Symbol dispatch entirely (spec deviation).

### Not protected (intentionally, any phase)

```js
userObj.method()          // unknown receiver type → normal lookup
subclass.push(x)          // subclass map ≠ initial JSArray map → normal lookup
proxy.get(key)            // proxy → normal lookup
obj[Symbol.iterator]()    // on non-builtin receiver → normal lookup
eval(code)                // dynamic code → never trusted
```

### await

Already safe: V8 implements `await` via internal C++ builtins (`PerformPromiseThen`). Not affected by `Promise.prototype.then` mutations.

## Migration Classifier

Not every primordial call site can be trivially replaced with plain JS. The following classifier helps determine which call sites are safe to migrate:

### Class A: Internally-created receivers — Safe rewrite

The receiver is created within Node.js internal code and never escapes to userland before use.

```js
// Class A: safe — arr is created internally, guaranteed JSArray
const arr = [];
arr.push(item);  // receiver map is initial JSArray map ✔

// Class A: safe — map created internally
const cache = new Map();
cache.set(key, value);  // receiver map is initial Map map ✔
```

**Migration:** Direct rewrite, remove primordial wrapper. No additional verification needed beyond confirming receiver origin.

### Class B: User-provided receiver — Semantic review required

The receiver comes from user code but the function expects it to be an ordinary builtin instance.

```js
// Class B: user passes array, internal code operates on it
function processItems(items) {
  // items is user-provided, but expected to be a plain Array
  items.push(newItem);  // works IF items has initial JSArray map
}
```

**Critical semantic difference from primordials:** With current primordials, `ArrayPrototypePush(items, newItem)` calls the **original** `Array.prototype.push` even if `items` is a subclass with an overridden `push`. After rewrite to `items.push(newItem)`, if `items` is a subclass, the IC falls through to normal lookup and calls the **subclass override** — not the original.

This is NOT "original semantics preserved." It is **normal JS semantics** for subclasses, which differs from primordial semantics.

**Migration:** Class B is safe to rewrite only when:
- Node.js wants normal JS behavior for subclass/proxy receivers (the common case for public-facing APIs), OR
- Node.js already validates/rejects non-plain receivers before operating on them.

Each Class B call site requires explicit semantic review. Add tests covering subclass/proxy edge cases.

### Class C: Subclass / proxy / array-like — Keep primordials

The code explicitly handles or expects non-standard receivers where the protection semantics differ.

```js
// Class C: must use primordials — receiver may be array-like
function toArray(arrayLike) {
  // arrayLike could be Arguments, NodeList, etc.
  // Map check will fail → IC falls through to normal lookup
  // If the original behavior relied on calling the original builtin on an
  // array-like receiver, keep primordials — IC will not resolve for non-initial maps.
  return ArrayPrototypeSlice(arrayLike);
}

// Class C: must use primordials — observable Symbol dispatch needed
function matchAll(str, regexp) {
  // regexp may have custom Symbol.match — must use current value
  return RegExpPrototypeExec(regexp, str);
}
```

**Migration:** Do not rewrite. Keep existing primordial calls. These are the ~20% that remain after the 80% automated migration.

### Classification decision tree

```
Is receiver created internally?
  YES → Class A (safe rewrite)
  NO →
    Is receiver expected to be a plain builtin instance?
      YES →
        Does the operation involve Symbol hooks (species, iterator, match, split, replace)?
          NO → Class B (rewrite with tests)
          YES → Class C (keep primordials)
      NO → Class C (keep primordials)
```

## Observable Semantics

The `use_initial_intrinsics` mode creates observable differences from both normal JS and current primordials. These must be documented for Node.js contributors:

### Identity comparisons

```js
// In trusted code:
const fn = arr.push;  // IC resolves to initial Array.prototype.push
// Even if userland did: Array.prototype.push = myPush

fn === Array.prototype.push;  // FALSE (if .push was patched)
fn === <original push>;       // TRUE (from native context)
```

This is the same behavior as primordials: `ArrayPrototypePush !== Array.prototype.push` after patching.

### Property access as value (not call)

```js
// In trusted code:
const method = arr.push;  // Returns initial push function
method(item);             // Calls initial push (but `this` is undefined in strict mode)
```

The IC resolves property loads, not just calls. The resolved value IS the initial intrinsic, regardless of how it's used.

**Warning:** `method.call(arr, item)` is only protected if `Function.prototype.call` is also in the trusted intrinsic whitelist. If userland patches `Function.prototype.call`, the `.call` lookup uses normal IC resolution. `Function.prototype.call/apply/bind` are NOT included in MVP v0 — they are a hot and complex zone. Internal code should prefer direct call syntax (`arr.push(item)`) over extracting methods.

**Migration rule:** Rewrite primordial prototype calls to method syntax only when the call remains syntactically direct, e.g. `ArrayPrototypePush(arr, x)` → `arr.push(x)`. Do not rewrite into extracted-method + `.call` patterns unless `Function.prototype.call/apply` is also handled or the code intentionally wants normal semantics. Codemods must not produce intermediate forms like `const push = arr.push; push.call(arr, x)`.

### typeof / instanceof

```js
// In trusted code:
typeof Object.keys;     // "function" — initial function from native context
Object.keys instanceof Function;  // true — it's the real initial function
```

No surprises here — initial intrinsics are real Function objects.

### Reflect.ownKeys / property enumeration from trusted code

Property enumeration on objects whose prototypes are patched will still show patched properties via normal prototype walk. The IC-level resolution only affects **direct property loads by name on known receivers**, not reflective operations like `Object.getOwnPropertyNames` or `for...in`.

## Primitive Receivers

For MVP v0, only String primitive receivers are relevant. Number/Boolean primitive methods are out of scope unless later added to the whitelist.

String primitive values are NOT JSString (heap objects with maps). They are tagged pointers. The IC receiver map check does **not** apply directly to them.

### How it works for primitives

When you write `str.slice(0, 5)` where `str` is a primitive string:

1. V8 performs auto-boxing conceptually, but in practice the LoadIC for String primitives checks the **string instance type** (ONE_BYTE_INTERNALIZED_STRING_TYPE, etc.), not a heap object map.
2. For the IC-level intrinsic resolution, we need a separate check:

```cpp
// In LoadIC for trusted scripts, primitive string receiver:
if (shared_function_info->use_initial_intrinsics() &&
    receiver->IsString() &&  // primitive string check
    IsInitialStringMethod(property_name)) {
  return native_context->get(intrinsic_slot);
}
```

3. This adds ~10–15 lines to the IC path specifically for primitive string/number methods.

### MVP impact

For the MVP, String methods (`slice`, `indexOf`, `trim`) operate on primitive strings. The IC handler must check both:
- Heap object receivers (JSString wrapper — rare in practice)
- Primitive string receivers (common case)

This is already how V8's existing String IC fast paths work — they check `IsString()` before map comparison. The intrinsic resolution piggybacks on this existing check.

## Compilation Cache and Dynamic Code

### Code cache (bytecode cache) requirements

When V8 caches compiled bytecode for internal modules (startup snapshot, code cache), the `use_initial_intrinsics` flag must be part of the cache key:

```cpp
// Cache key includes the flag to prevent cross-contamination:
// [source_hash, use_initial_intrinsics, language_mode, ...]
```

A module compiled with `use_initial_intrinsics = true` must never be served from cache to a context expecting `false`, and vice versa. Cached data produced with one value of the flag must be rejected when consumed with the other value. In practice, Node.js internal modules are always compiled with the flag, so the cache is cleanly partitioned.

### Dynamic code: eval / new Function / vm

Code generated dynamically via `eval()`, `new Function()`, or `vm.compileFunction()` is **never** trusted, regardless of the calling context:

```cpp
// In Parser / CompileInfo:
if (is_eval || is_dynamic_function) {
  shared_function_info->set_use_initial_intrinsics(false);  // ALWAYS
}
```

This is a hard compile-time policy invariant. It does not create a sandbox, but prevents trusted-intrinsic semantics from leaking to dynamic/user-provided code. Even if `eval` is called inside a `node:internal/*` module, the evaluated code uses normal semantics.

### Lazy function propagation

When a trusted module defines inner functions (closures, callbacks), they inherit the `use_initial_intrinsics` flag from their enclosing script:

```js
// node:internal/streams/writable.js — trusted script
function write(chunk) {         // use_initial_intrinsics = true (from script)
  const arr = [];
  process.nextTick(() => {      // closure inherits flag
    arr.push(chunk);            // IC resolves to initial push ✔
  });
}
```

This is handled naturally: `SharedFunctionInfo` is created per-function at parse time, and the script-level flag propagates to all `SharedFunctionInfo` objects within that script. Lazy-compiled functions pick up the flag when their `SharedFunctionInfo` is eventually compiled.

## Node.js Patch

### Passing the flag (~10 lines)

```cpp
// src/node_contextify.cc or src/module_wrap.cc
v8::ScriptOriginOptions options(
  /* is_shared_cross_origin */ false,
  /* is_opaque */              false,
  /* is_wasm */                false,
  /* is_module */              is_esm,
  /* use_initial_intrinsics */ is_internal  // NEW
);
v8::ScriptOrigin origin(isolate, filename, 0, 0, options);
```

### Determining "internal" (~5 lines)

```cpp
bool is_internal = filename.starts_with("node:internal/")
                || filename.starts_with("node:_")
                || kBootstrapModules.contains(filename);
```

Security: `node:internal/*` modules are already blocked from userland `require`/`import`. Must verify no bypass via `vm.compileFunction`, `--require`, or custom ESM loaders.

## What the Developer Sees

### Before (primordials)

```js
'use strict';
const {
  ArrayPrototypePush,
  ObjectKeys,
  JSONParse,
  StringPrototypeSlice,
} = primordials;

function processData(arr, config) {
  const keys = ObjectKeys(config);
  for (let i = 0; i < keys.length; i++) {
    ArrayPrototypePush(arr, StringPrototypeSlice(keys[i], 0, 10));
  }
  return JSONParse(serialize(arr));
}
```

### After (plain JavaScript)

```js
'use strict';

function processData(arr, config) {
  const keys = Object.keys(config);
  for (let i = 0; i < keys.length; i++) {
    arr.push(keys[i].slice(0, 10));
  }
  return JSON.parse(serialize(arr));
}
```

This example uses only MVP v0-protected operations: `Object.keys`, `arr.push`, `String.prototype.slice`, `JSON.parse`. Both versions call original builtins regardless of userland monkey patching. The difference: compiler-level resolution vs runtime wrappers.

## Comparison with Other Approaches

### Primordials (current)

Save references to original builtins at startup. Use `uncurryThis` wrappers. Safe and deterministic, but unreadable code and performance overhead from indirect calls and megamorphic ICs. Works across all tiers.

### Skip prototype dependencies (previous version of this proposal)

Skip `CompilationDependencies` for internal code. Only works after optimization (Maglev/TurboFan). Behavior depends on tiering state. Does not cover globals. Too broad — disables dependencies for all lookups, not just builtins.

### CoreOps (JS wrappers)

Replace primordial names with cleaner API (`CoreArray.push`). Same implementation underneath — saved references + indirect calls. Primordials renamed. Same overhead.

### CoreOps (C++ bindings)

Built-in operations as C++ functions via `internalBinding`. No prototype dependency, but JS→C++ boundary cost (~50–100ns per call vs ~2–5ns inlined). Too slow for hot paths.

### Frozen context

Separate V8 context with frozen prototypes. Performance excellent, but cross-context identity issues, doubled memory, complex debugging. ~1000+ lines in Node.js.

### Initial intrinsics resolution (this proposal)

IC-level resolution against native context initial intrinsics. Works across all tiers. Whitelist-based — only known builtins on known receiver maps. PoC ~300–700 lines, upstream MVP ~900–1700+ lines in V8.

### Summary table

```
                          Perf  DX   All tiers  Whitelist  Context switch  V8 patch size
Primordials (current)     ⚠️    ❌    ✅          ✅          no              none
Skip dependencies (v2)    ✅    ✅    ❌          ❌          no              ~100-150
CoreOps (JS)              ⚠️    ⚠️    ✅          ✅          no              none
CoreOps (C++)             ❌    ⚠️    ✅          ✅          no              none
Frozen context            ✅    ✅    ✅          N/A        YES             ~1000+ (Node)
Initial intrinsics        ✅    ✅    ✅          ✅          no              PoC ~300-700
                                                                            MVP ~900-1700+
```

## Why V8 Team May Accept

1. **Semantic, not optimization** — this is not "skip a deopt guard." It is a well-defined mode: "resolve known intrinsics from native context initial slots." Consistent across all tiers.

2. **Whitelist-based** — only known builtins on known receiver maps. Does not affect arbitrary property lookups. No correctness holes for userland objects.

3. **Precedent** — Torque/CSA builtins already call initial intrinsics directly from native context slots. This extends the same mechanism to embedder JavaScript.

4. **Uses existing infrastructure** — many required native context intrinsic slots already exist. The patch reuses IC infrastructure but should be implemented via specialized trusted handlers or miss paths to avoid adding cost to the common untrusted fast path.

5. **Useful for V8 embedders** — primarily Node.js; potentially Deno, Bun, Electron, and other embedders with trusted internal JavaScript, subject to their own security models.

## Why V8 Team May Reject

1. **IC path complexity** — adding a branch in the hot LoadIC path. Every microsecond matters. The implementation must avoid an unconditional branch/load in the common fast path. The target is **zero overhead for non-trusted scripts** — e.g., by using a separate handler kind or IC miss path rather than checking an SFI bit on every LoadIC.

2. **Maintenance** — intrinsics table must be maintained as new builtins are added. V8 team must accept this as a maintained surface.

3. **Scope** — the upstreamable MVP is likely 800–1500+ lines including tests, because it touches IC, feedback, compiler, snapshot/cached data, API documentation, and public API. Multiple reviewers needed.

4. **"Optimize primordials instead"** — V8 could alternatively teach TurboFan to recognize `uncurryThis(Array.prototype.push)(arr, x)` as a direct builtin call. Doesn't solve DX, but avoids new API surface.

## Risks

1. **IC performance** — the additional check in LoadIC must not slow down normal (non-trusted) code. The implementation should avoid adding an unconditional SFI load to the common LoadIC hot path. Possible strategies:
   - Encode the mode in the FeedbackVector or feedback nexus, so the IC slot itself selects the handler kind.
   - Generate a separate trusted-intrinsic IC handler kind only for trusted scripts.
   - Route trusted scripts through a separate IC miss path, keeping the normal userland fast path completely unchanged.
   The goal: zero cost for non-trusted code by handler specialization, not runtime flag checks.

2. **Incomplete coverage** — the MVP whitelist covers ~15 entries. Later phases may expand to ~50–100 common intrinsics. Edge cases (Symbol.species, RegExp symbol hooks, exotic descriptors) may still need primordials. Pragmatic: start with 80/20, keep primordials as fallback for rare cases.

3. **Security** — same as before: must prove `use_initial_intrinsics` cannot be set for userland code. Node already blocks `node:internal/*` access. Must verify `vm.compileFunction`, `--require`, `--loader` paths. Additionally: the trust boundary is about **who can compile code with the flag**, not about what data trusted code processes. Trusted Node internals still process user-provided values (e.g., `fs.writeFile(path, data)`, `stream.write(chunk)`), so each migration must audit receiver provenance and fallback behavior. Must also verify behavior with `--expose-internals`, `internal/test/binding`, worker threads, and vm contexts.

4. **V8 team rejects** — fallback to automated primordials (codemod) or V8 primordials optimization.

---

## Implementation Plan

### Phase 0: Proof of Concept

**Goal:** working prototype in V8 fork + benchmarks.

**Steps:**

1. Fork V8 from current stable
2. **Audit NativeContext slots** — verify which desired MVP operations already have stable context slots (e.g., `Context::ARRAY_PUSH_INDEX`). Document missing entries that need new slots.
3. Add `use_initial_intrinsics` to `ScriptOriginOptions`
4. Add bit to `SharedFunctionInfo`
5. Create initial intrinsics table:
   - **PoC v0.1** (~12 entries): Array push/pop/shift/unshift, Map get/set/has, Set add/has, Object.keys, Array.isArray, JSON.parse. First behavioral commit should start with Array.prototype.push only; additional mutators added after IC mechanism is proven.
   - **PoC v0.2** (follow-up commit): String slice/indexOf/trim (requires primitive receiver IC path)
6. Implement IC-level intrinsic resolution for LoadIC:
   - Check SFI flag
   - Check receiver map is initial builtin instance map
   - Lookup in intrinsics table
   - Return from native context slot
7. Build Node.js against patched V8
8. Pass `use_initial_intrinsics = true` for `node:internal/*`
9. Remove primordials from 2–3 hot modules (streams, buffer)
10. Run benchmarks and correctness tests:
   - Throughput: `node benchmark/streams/`, `node benchmark/fs/`
   - Startup: `time node -e "1"`
   - Correctness: monkey patch Array.prototype.push, verify internal code still calls original
   - Correctness: monkey patch under `--jitless`, verify same behavior
   - Correctness: pass userland object to internal function, verify normal lookup
   - Correctness: static aliasing (`const O = Object; Object.keys = hacked; O.keys({a:1})` → initial)
   - Correctness: extracted static (`const keys = Object.keys; Object.keys = hacked; keys({a:1})` → initial)
   - Correctness: polymorphic site (initial array + subclass array → both correct)
   - Correctness: `Function.prototype.call` patch does NOT affect direct `arr.push(x)` calls
   - Correctness: `eval(...)` inside trusted module uses normal semantics
   - Correctness: global namespace replacement (`globalThis.JSON = { parse: hacked }`) does not affect trusted `JSON.parse`
   - Correctness: lexical shadowing (`function f(Object) { Object.keys({}) }`) uses parameter, not initial
   - Overhead: untrusted code LoadIC benchmarks show zero regression
   - Overhead: snapshot size and native context size delta

**Deliverable:** branch with working code + benchmark + correctness results.

**Go/no-go:** if no measurable perf improvement OR correctness issues under `--jitless` OR non-trivial untrusted-code LoadIC overhead OR unacceptable snapshot/native-context size growth → rethink approach.

**PoC commit strategy:** For the first PoC commit, start without primitive receivers (String). Prove the core mechanism on object/map/global resolution first. Add String primitive IC support as a follow-up commit.

### Phase 1: Design Doc + V8 Discussion

**Goal:** V8 team feedback before submitting CL.

**Steps:**

1. Write design doc (Google Docs):
   - Problem statement with primordials overhead numbers
   - Exact mechanism: IC-level initial intrinsic resolution
   - Intrinsics table specification
   - Benchmark results from Phase 0
   - Correctness verification: all tiers, `--jitless`, monkey patching
   - Security analysis
   - Comparison with alternatives
2. File bug in V8 tracker (issues.chromium.org)
   - Component: `Blink>JavaScript>Runtime` or `Blink>JavaScript>Compiler`
   - Title: "Add use_initial_intrinsics option for embedder trusted scripts"
3. Engage V8 reviewers:
   - IC/runtime team for LoadIC changes
   - Compiler team for Maglev/TurboFan inlining
   - Shu-yu Guo for spec compliance
4. Coordinate with Joyee Cheung (Node ↔ V8 bridge)

**Risk checkpoint:** V8 feedback determines next step.

### Phase 2: V8 CL

**Goal:** land the patch.

**Files:**
```
include/v8-script.h                     — ScriptOriginOptions bit
src/objects/shared-function-info.h      — SFI bit
src/objects/initial-intrinsics-table.h  — whitelist (new file)
src/ic/ic.cc                            — LoadIC intrinsic resolution
src/ic/ic.cc                            — LoadGlobalIC intrinsic resolution
src/objects/feedback-vector*            — kInitialIntrinsicLoad handler metadata
src/codegen/ or compilation-cache       — cache key / cached-data compatibility
src/compiler/compilation-dependencies.cc — skip dependency for intrinsic sites
src/maglev/maglev-graph-builder.cc      — intrinsic inline
src/snapshot/                           — flag preservation
test/cctest/test-ic.cc                  — IC tests
test/mjsunit/initial-intrinsics.js     — JS correctness tests
```

**Tests:**
- Compile with `use_initial_intrinsics`, monkey patch prototype → original builtin called
- Same test under `--jitless` → same result
- Compile without flag, monkey patch → patched version called
- Userland object in trusted code → normal lookup
- Subclass in trusted code → normal lookup
- Proxy in trusted code → normal lookup
- Snapshot: flag survives serialization
- IC: feedback vector shows intrinsic handler

**Review:** OWNERS (IC team + compiler team), Chrome security review, CQ.

### Phase 3: Node.js PR — Enable the Flag

1. Update V8 to version with `use_initial_intrinsics`
2. Pass flag for `node:internal/*` modules
3. Add Node-side tests for correctness
4. CI: full test suite

### Phase 4: Migrate Eligible Primordial Call Sites

For each internal module, by hotness:

1. **Streams** (`lib/internal/streams/*`)
2. **Buffer** (`lib/internal/buffer.js`)
3. **FS** (`lib/internal/fs/*`)
4. **Net/HTTP** (`lib/internal/net.js`, `lib/internal/http.js`)
5. **Timers** (`lib/internal/timers.js`)
6. **Remaining** — all other `lib/internal/`

Each module = separate PR:
- Replace `ArrayPrototypePush(arr, x)` → `arr.push(x)`
- Replace `ObjectKeys(obj)` → `Object.keys(obj)`
- Remove `const { ... } = primordials;`
- Run module tests + full CI

### Phase 5: Reduce Primordials Surface

1. Verify all Class A and approved Class B call sites are migrated
2. Keep primordials/Safe* helpers for Class C and complex spec operations (Symbol.species, RegExp hooks, thenable assimilation, array-like receivers)
3. Optionally consolidate remaining primordials into a smaller `internal/safe_intrinsics` module
4. Delete only unused primordial entries — not necessarily the entire primordials system
5. Full primordials removal is possible only if future phases (A, B, C) cover all remaining operations

---

## Recommended Rollout Strategy

To minimize pressure on V8 reviewers, prefer a data-first approach:

1. **Node/V8 downstream experiment** behind a build flag — no public API commitment yet.
2. **Collect benchmark + correctness data** from PoC (Phase 0).
3. **Present results** before requesting public API — demonstrate measurable benefit and zero overhead for untrusted code.
4. If V8 rejects public API, keep as Node downstream experiment or pursue Fallback A/B.

This separates "does the mechanism work?" from "should it be a V8 public API?" and gives V8 team confidence before committing to maintenance.

---

## Fallback Plan

If V8 rejects `use_initial_intrinsics`:

**Fallback A: Optimize existing primordials in V8**
- Teach V8 to recognize `uncurryThis(Builtin.prototype.method)(receiver, args)` as a direct builtin call
- No semantic change, no new API — purely optimization
- Keeps primordials in Node, but removes perf overhead
- Smaller V8 patch, easier to accept
- **Limitation:** does not address DX. Contributors must still learn and follow the primordials discipline documented in Node's contributing guidelines (unsafe iteration patterns, descriptor caveats, RegExp/Promise hooks, etc.). The cognitive burden remains. Initial intrinsics would move that knowledge into the engine, where receiver maps, native context slots, and IC feedback are already available — replacing Node-level conventions with V8-level mechanisms.

**Fallback B: Build-time codemod**
- Developers write plain JS in `lib/internal/*`
- Build step transforms `arr.push(x)` → `ArrayPrototypePush(arr, x)` at Node.js build time
- Source maps for debugging
- DX improves, performance stays the same
- No V8 changes needed

**Fallback C: Keep primordials**
- Accept status quo
- Incrementally reduce primordials usage in non-hot paths

Work from Phase 0 (benchmarks, module audit) is useful regardless of path.

---

## Key People

- **Joyee Cheung** (Node + V8) — startup snapshot, Node ↔ V8 bridge
- **Leszek Swirski** (V8 compiler) — compilation pipeline
- **Toon Verwaest** (V8 IC/runtime) — inline cache changes
- **Shu-yu Guo** (V8 + TC39) — spec compliance
- **Yagiz Nizipli** (Node.js performance) — benchmarks
- **Matteo Collina** (Node.js TSC, streams) — DX champion

## How to Start

1. Fork V8
2. Audit NativeContext slots for PoC v0.1 operations
3. Create intrinsics table for PoC v0.1 (~12 entries), starting with Array.prototype.push
4. Patch LoadIC to resolve from native context for trusted scripts
5. Build Node against fork
6. Remove primordials from `lib/internal/streams/writable.js`
7. Run benchmarks + correctness tests (including `--jitless`)
8. If results are good → expand to full MVP v0 → write design doc → file V8 bug

---

## Supporting Documents

| Document | Purpose |
|---|---|
| [docs/poc-plan.md](poc-plan.md) | Phase 0 commit sequence, correctness test matrix, go/no-go criteria |
| [docs/benchmarks.md](benchmarks.md) | Benchmark configurations, scripts, result tables |
| [docs/security.md](security.md) | Security invariants, bypass scenarios, Node migration checklist |
| [notes/native-context-audit.md](../notes/native-context-audit.md) | V8 NativeContext slot audit for MVP targets |
