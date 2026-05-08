# Trusted Initial Intrinsics for V8 Embedders — Full Proposal

## Background

V8 embedders such as Node.js need a way to execute internal, trusted scripts that are immune to user-land monkey patching of built-in objects (e.g. `Array.prototype.push`, `Object.keys`). Today, Node.js solves this via **primordials** — a large frozen snapshot of built-in values captured at startup. While effective, primordials impose a significant developer-experience (DX) cost (verbose, unfamiliar syntax) and can hurt performance in certain paths due to indirection and the inability for the JIT to inline well-known intrinsics.

## Problem Statement

1. **Monkey patching risk** — user code runs in the same realm as Node internals and can replace any built-in at any time.
2. **Primordials DX cost** — contributors must write `primordials.ArrayPrototypePush(arr, v)` instead of `arr.push(v)`.
3. **Performance regression** — indirect calls via primordials are harder to optimize compared to inline calls to known V8 built-in intrinsics.

## Proposal

Introduce a V8 API flag (or a dedicated script-compilation option) that marks a script as **trusted**. When a trusted script loads a property from a global or prototype object that appears in a built-in whitelist, V8 resolves it to the *initial* (pre-startup) intrinsic value rather than the current runtime value, bypassing any user-land overrides.

### Key Design Points

- **Opt-in per script** — only scripts explicitly tagged as trusted (e.g. Node.js internal modules) get initial-intrinsic resolution.
- **Whitelist-based** — only a curated set of properties (see MVP) use initial resolution; everything else follows normal lookup semantics.
- **No new realm** — trusted scripts still share the same realm for interoperability; only the named property loads are redirected.
- **NativeContext anchor** — initial values are read once from `NativeContext` slots populated during V8 bootstrap, before any user code runs.

### Whitelist (MVP)

| Property | NativeContext slot |
|---|---|
| `Array.prototype.push` | `kArrayPrototypePushIndex` |
| `Object.keys` | `kObjectKeysIndex` |
| `Array.isArray` | `kArrayIsArrayIndex` |
| `JSON.parse` | `kJSONParseIndex` |

## Alternatives Considered

| Alternative | Drawback |
|---|---|
| Primordials (status quo) | DX cost, JIT opacity |
| Separate realm / `vm.runInNewContext` | Interop friction, memory overhead |
| Proxy-based shadow globals | Runtime overhead, limited JIT benefit |
| SES / Compartments | Heavy machinery, spec-level dependency |

## Open Questions

1. Should the whitelist be configurable by the embedder at startup, or hard-coded in V8?
2. How should `Reflect.get` and computed property access interact with trusted resolution?
3. Does this require changes to the V8 `Context` snapshot, or can it be layered on top?

## References

- [Node.js primordials](https://github.com/nodejs/node/blob/main/lib/internal/per_context/primordials.js)
- [V8 NativeContext](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/objects/contexts.h)
- [TC39 SES proposal](https://github.com/tc39/proposal-ses)
