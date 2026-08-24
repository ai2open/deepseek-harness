# Agent Note: Browser unary RPC mints rpcIds without the secure-context-only crypto.randomUUID

Status: implemented

English | [中文](2026-08-24-browser-unary-rpcid-insecure-origin.zh.md)

## Problem

The `/api` trust fence deliberately admits plain-HTTP non-loopback authorities — deployment-derived LAN IP literals and declared `trustedHosts` — so serving the web GUI to a LAN or NAT-mapped address is a supported deployment. Browsers expose `crypto.randomUUID()` only in secure contexts (HTTPS or localhost); on those admitted plain-HTTP origins the function is `undefined`. `AbstractApiClient.mintRpcId()` called it directly and `WebApiClient` did not override it, so every unary RPC — `workspace.list`, `workspace.create`, `host.listDirectory`, and the rest — threw `TypeError: crypto.randomUUID is not a function` while minting its rpcId, before any network traffic. The GUI was unusable from exactly the deployments the fence admits. The connection package already owned `randomUuid()` (backed by `crypto.getRandomValues()`, which browsers expose on insecure origins) for the generic `rpc.call` channel, with secure-context regression coverage; the unary carrier path was missed.

## Decision

`WebApiClient` overrides `mintRpcId()` and mints with the connection package's `randomUuid()` helper, so the browser platform subclass carries the insecure-origin form. The `AbstractApiClient` base keeps `crypto.randomUUID()` for its host-side and in-process consumers (Node ≥ 19 and secure contexts), unchanged. Regression coverage in `client-apply.client.spec.ts` stubs `globalThis.crypto` down to `getRandomValues` and asserts a unary `workspace.list` call reaches the wire with a v4 rpcId — the mirror of the existing `rpc.call` secure-context case. The mechanism of record for rpcId ownership stays with the [GUI layering and RPC protocol note](2026-07-19-gui-layering-and-rpc-protocol.md): the initiator mints, and for the browser initiator the mint must not assume a secure context.

## Alternatives considered

- **Move the fallback into `AbstractApiClient.mintRpcId()`**: works everywhere (Node ≥ 15 has `crypto.getRandomValues()`), but the secure-context constraint is browser-specific, and the browser-safe helper already lives in the connection package beside its other consumer; the override keeps the fix at the layer that knows the constraint instead of spreading browser concerns into the host-owned base.
- **Feature-detect `crypto.randomUUID` and fall back inline**: duplicates the v4-from-`getRandomValues` logic in a second place behind a silent branch; a single named helper with direct coverage is stricter.
- **Declare plain-HTTP LAN serving unsupported and require HTTPS or localhost**: contradicts the trust fence's admitted set and the CLI's own LAN-URL derivation; the product intends these deployments to work.

## Consequences

The two faces mint differently — the host/in-process base via the Web API, the browser subclass via `getRandomValues` — one implementation per platform face, each directly covered. Plain-HTTP LAN and NAT-mapped deployments can drive the full GUI. Known gap, deliberately out of scope: `packages/client/ui-conversation/src/client/service.ts` still calls `crypto.randomUUID()` for draft attachment ids, so attaching a file on an insecure origin fails the same way; it is a separate package with its own coverage and is left to a follow-up.
