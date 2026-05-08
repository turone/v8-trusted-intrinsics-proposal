# Trusted Initial Intrinsics for V8 Embedders

## Problem

Node.js internal modules use primordials for protection from monkey patching, but this hurts DX and can hurt performance.

## Proposal

Trusted internal scripts can opt into initial-intrinsic resolution for a whitelist of builtin property loads.

## MVP

- `Array.prototype.push`
- `Object.keys`
- `Array.isArray`
- `JSON.parse`

## Status

Design draft. PoC in progress.

## Links

- [Full proposal](docs/proposal.md)
- [PoC plan](docs/poc-plan.md)
- [Benchmark results](docs/benchmarks.md)
- [Security considerations](docs/security.md)
