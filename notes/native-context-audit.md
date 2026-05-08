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

## Findings

> **Status:** Pending PoC Phase 2 completion.

No slots have been verified against a live V8 build yet. This document will be updated as the PoC progresses.

## References

- `v8/src/objects/contexts.h` — slot definitions
- `v8/src/init/bootstrapper.cc` — slot population during startup
- `v8/src/runtime/runtime-object.cc` — runtime functions that may mutate slots
