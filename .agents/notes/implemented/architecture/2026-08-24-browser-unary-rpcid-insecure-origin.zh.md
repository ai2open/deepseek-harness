# Agent Note: Browser unary RPC mints rpcIds without the secure-context-only crypto.randomUUID

Status: implemented

[English](2026-08-24-browser-unary-rpcid-insecure-origin.md) | 中文

## Problem

`/api` 信任栅栏刻意放行普通 HTTP 的非回环 authority——部署推导的 LAN IP 字面量与声明的 `trustedHosts`——因此把 web GUI 服务到 LAN 或 NAT 映射地址是受支持的部署形态。浏览器只在安全上下文（HTTPS 或 localhost）暴露 `crypto.randomUUID()`；在这些被放行的普通 HTTP 来源上该函数是 `undefined`。`AbstractApiClient.mintRpcId()` 直接调用它，而 `WebApiClient` 没有覆写，于是每个 unary RPC——`workspace.list`、`workspace.create`、`host.listDirectory` 等等——在铸造 rpcId 时就抛出 `TypeError: crypto.randomUUID is not a function`，连网络流量都未发出。栅栏放行的部署恰好全都用不了 GUI。连接包已经拥有基于 `crypto.getRandomValues()`（浏览器在非安全来源也暴露）的 `randomUuid()`，供通用 `rpc.call` 通道使用并配有安全上下文回归覆盖；unary 载体路径被遗漏了。

## Decision

`WebApiClient` 覆写 `mintRpcId()`，用连接包的 `randomUuid()` 帮助函数铸造 rpcId，由浏览器平台子类携带非安全来源形态。`AbstractApiClient` 基类为其宿主侧与进程内消费者（Node ≥ 19 及安全上下文）保留 `crypto.randomUUID()`，不变。`client-apply.client.spec.ts` 中的回归覆盖把 `globalThis.crypto` 裁到只剩 `getRandomValues`，断言一次 unary `workspace.list` 调用携带 v4 rpcId 到达线上——与既有 `rpc.call` 安全上下文用例互为镜像。rpcId 归属的机制记录仍在 [GUI 分层与 RPC 协议笔记](2026-07-19-gui-layering-and-rpc-protocol.zh.md)：发起方铸造，而浏览器发起方的铸造不得假设安全上下文。

## Alternatives considered

- **把回退挪进 `AbstractApiClient.mintRpcId()`**：到处可用（Node ≥ 15 即有 `crypto.getRandomValues()`），但安全上下文约束是浏览器特有的，且浏览器安全帮助函数已经住在连接包里、挨着它的另一个消费者；覆写把修复留在了解该约束的层，而不是把浏览器关注点扩散进宿主侧基类。
- **特性探测 `crypto.randomUUID` 并内联回退**：会在第二个位置、以一条静默分支复制 v4-from-`getRandomValues` 逻辑；有直接或覆盖的单一命名帮助函数更严格。
- **宣布普通 HTTP LAN 服务不受支持、要求 HTTPS 或 localhost**：与信任栅栏的放行集合及 CLI 自己的 LAN URL 推导相矛盾；产品意图让这些部署可用。

## Consequences

两个 face 的铸造方式不同——宿主/进程内基类走 Web API，浏览器子类走 `getRandomValues`——每个平台 face 一个实现，各有直接覆盖。普通 HTTP 的 LAN 与 NAT 映射部署可以驱动完整 GUI。已知缺口，刻意留在范围外：`packages/client/ui-conversation/src/client/service.ts` 仍为附件草稿 id 直接调用 `crypto.randomUUID()`，在非安全来源上添加附件会以同样方式失败；它属于另一个包、有自己的覆盖，留给后续处理。
