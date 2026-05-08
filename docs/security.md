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
