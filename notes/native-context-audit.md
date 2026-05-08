# Native Context Audit

Audit of `NativeContext` slots in V8 relevant to the Trusted Initial Intrinsics proposal.

Source reference: `v8/src/objects/contexts.h` (V8 12.x).

## Purpose

Map each MVP whitelisted property to its NativeContext slot so the PoC implementation can emit direct slot loads instead of runtime property lookups.

## MVP Slot Mapping

| JS Property | Context macro | Slot index | Value type |
|---|---|---|---|
| `Array.prototype.push` | `ARRAY_PUSH_INDEX` | TBD | `JSFunction` |
| `Object.keys` | `OBJECT_KEYS_INDEX` | TBD | `JSFunction` |
| `Array.isArray` | `ARRAY_IS_ARRAY_INDEX` | TBD | `JSFunction` |
| `JSON.parse` | `JSON_PARSE_INDEX` | TBD | `JSFunction` |

> **TODO:** Fill in exact slot indices after grepping `v8/src/objects/contexts.h` on the target V8 tag.

## Audit Methodology

1. Clone V8 at the target tag.
2. Run: `grep -n 'ARRAY_PUSH\|OBJECT_KEYS\|ARRAY_IS_ARRAY\|JSON_PARSE' src/objects/contexts.h`
3. Cross-check that each macro expands to a unique integer index.
4. Verify that the slot is populated in `Bootstrapper::Genesis` before any user script runs.
5. Confirm the slot is read-only after bootstrap (no re-assignment in `runtime/` files).

## Additional Candidates for Future Expansion

| JS Property | Notes |
|---|---|
| `Function.prototype.call` | Very hot in internal code |
| `Function.prototype.apply` | Hot in argument forwarding |
| `String.prototype.replace` | Used heavily in URL/path processing |
| `Promise.resolve` | Async internals |
| `Symbol.iterator` | Protocol bootstrapping |
| `Object.defineProperty` | Module system internals |
| `Object.freeze` | Primordials-like hardening |
| `Object.create` | Prototype chain setup |

## Per-Slot Questions

For each MVP intrinsic, answer the following before assigning a slot in the implementation:

1. Where is the initial function object created (which file in `v8/src/init/` or `v8/src/bootstrapper.cc`)?
2. Is there an existing `Context::*_INDEX` macro for it in `v8/src/objects/contexts.h`?
3. Is the slot reachable from NativeContext without a prototype chain walk?
4. Is the slot populated in `Bootstrapper::Genesis` before any user script can run?
5. Is the slot read-only after bootstrap (no re-assignment in `v8/src/runtime/`)?
6. Does the slot store the actual `JSFunction` object suitable for a property-load identity check?
7. Would adding a new slot be necessary? If so, what is the memory impact per context?
8. What tests are needed to verify the slot survives snapshot round-trips?

### Array.prototype.push

| Question | Answer |
|---|---|
| Created in | TBD |
| Existing Context macro | `ARRAY_PUSH_INDEX` — verify in `contexts.h` |
| Reachable without prototype walk | TBD |
| Populated before user scripts | TBD |
| Read-only after bootstrap | TBD |
| Stores JSFunction | TBD |
| Needs new slot | TBD |
| Memory impact | TBD |

### Object.keys

| Question | Answer |
|---|---|
| Created in | TBD |
| Existing Context macro | `OBJECT_KEYS_INDEX` — verify in `contexts.h` |
| Reachable without prototype walk | TBD |
| Populated before user scripts | TBD |
| Read-only after bootstrap | TBD |
| Stores JSFunction | TBD |
| Needs new slot | TBD |
| Notes | Also needs initial Object constructor slot for global-load step |

### Array.isArray

| Question | Answer |
|---|---|
| Created in | TBD |
| Existing Context macro | `ARRAY_IS_ARRAY_INDEX` — verify in `contexts.h` |
| Reachable without prototype walk | TBD |
| Populated before user scripts | TBD |
| Read-only after bootstrap | TBD |
| Stores JSFunction | TBD |
| Needs new slot | TBD |

### JSON.parse

| Question | Answer |
|---|---|
| Created in | TBD |
| Existing Context macro | `JSON_PARSE_INDEX` — verify in `contexts.h` |
| Reachable without prototype walk | TBD |
| Populated before user scripts | TBD |
| Read-only after bootstrap | TBD |
| Stores JSFunction | TBD |
| Notes | Also needs initial JSON namespace object slot |

### Map.prototype.get / Map.prototype.set

| Question | Answer |
|---|---|
| Existing Context macro | TBD — `MAP_GET_INDEX` / `MAP_SET_INDEX` if they exist |
| Initial Map map | `initial_js_map_map()` in NativeContext — verify |
| Needs new slot | TBD |

### Set.prototype.add / Set.prototype.has

| Question | Answer |
|---|---|
| Existing Context macro | TBD — `SET_ADD_INDEX` / `SET_HAS_INDEX` if they exist |
| Initial Set map | `initial_js_set_map()` in NativeContext — verify |
| Needs new slot | TBD |

---

## If a Slot Does Not Exist

### Option A: Exclude from MVP

Use only operations that already have stable NativeContext slots. Simplest approach, smallest patch, lowest review risk.

**Pros:** no context layout changes, no snapshot impact, no memory per context.
**Cons:** some useful operations may be deferred.

### Option B: Add a new NativeContext slot

Add a new `Context::*_INDEX` entry for the missing operation.

**Pros:** direct access, stable identity, consistent with existing slot architecture.
**Cons:** changes context layout, affects snapshot format, adds memory per context (~8 bytes per slot), requires more review surface. Must be populated in `Bootstrapper::Genesis` and verified in snapshot tests.

### Option C: Reference via Builtin ID

Use a `Builtin::k*` enum value instead of a slot index, and resolve to the entry point at IC time.

**Pros:** no new slot, no memory impact.
**Cons:** may not preserve the actual `JSFunction` object identity required for property-load semantics. Suitable for call-target optimization but potentially incorrect if IC needs to return the actual function object for identity checks (`fn === Array.prototype.push` patterns).

---

## Open Questions

1. Does a trusted property load need to return the actual `JSFunction` object, or can it return a builtin callable? (Affects whether Option C is viable.)
2. How does identity of the returned value interact with `fn === Array.prototype.push` style checks in internal code?
3. Can feedback store a Builtin ID while LoadIC returns the actual `JSFunction` object?
4. Which existing `Context::*_INDEX` constants are stable enough across V8 versions that Node.js can safely depend on them?
5. What is the exact cost (bytes) of adding one new NativeContext slot across all context instances in a typical Node.js process?

## Findings

> **Status:** Pending PoC Phase 2 completion.

No slots have been verified against a live V8 build yet. This document will be updated as the PoC progresses.

## References

- `v8/src/objects/contexts.h` — slot definitions
- `v8/src/init/bootstrapper.cc` — slot population during startup
- `v8/src/runtime/runtime-object.cc` — runtime functions that may mutate slots
