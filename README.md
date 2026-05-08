# Trusted Initial Intrinsics for V8 Embedders

## Problem

Node.js internal modules use primordials for protection from monkey patching, but this hurts DX and can hurt performance. Preliminary measurements ([tshemsedinov/poc-node-isolate-internals](https://github.com/tshemsedinov/poc-node-isolate-internals)) show the primordials approach carries ~2× overhead vs. direct calls.

## Proposal

Trusted internal scripts can opt into initial-intrinsic resolution for a whitelist of builtin property loads. V8 resolves these loads against initial values from the native context, regardless of userland mutations.

## MVP

- `Array.prototype.push`
- `Object.keys`
- `Array.isArray`
- `JSON.parse`

## Scope

### What changes for trusted scripts

- Named property loads (`arr.push`, `Object.keys`, `JSON.parse`) on proven initial builtin receivers/constructors — resolved to NativeContext initial values at every tier, including `--jitless`.
- Global property loads for known built-in global names (`Object`, `Array`, `JSON`, etc.) — resolved to initial constructor/namespace.

### What does NOT change

- Userland code — no change in any behavior.
- Subclass instances — subclass map ≠ initial map → normal prototype chain lookup.
- Proxy objects — normal proxy semantics.
- Dynamic key access (`obj[dynamicKey]`, symbol access, `for...in`) — always normal.
- Argument semantics — proxy traps, getters, callbacks, revivers, coercions all apply normally.
- Dynamic code (`eval`, `new Function`, `vm.compileFunction`) — always untrusted regardless of calling context.
- Lexical bindings shadowing globals — always resolved normally.

## Status

Design draft. PoC in progress.

## Success Criteria

- Trusted code consistently calls the original intrinsic even after userland mutation, across all tiers including `--jitless`.
- Untrusted code behavior is unchanged in all cases.
- No measurable LoadIC regression for non-trusted code.
- Microbenchmark shows ≤1.1× overhead vs. unprotected direct calls (closing the ~2× primordials gap).
- Snapshot size and NativeContext size delta are acceptable.

## Links

- [Full proposal](docs/proposal.md)
- [PoC plan](docs/poc-plan.md)
- [Benchmark plan](docs/benchmarks.md)
- [Security considerations](docs/security.md)
- [NativeContext slot audit](notes/native-context-audit.md)
