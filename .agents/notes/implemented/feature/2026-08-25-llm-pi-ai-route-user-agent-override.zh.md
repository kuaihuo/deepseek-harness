# Agent Note: 路由声明的 `User-Agent` 覆盖 Harness 归因头

Status: implemented

[English](2026-08-25-llm-pi-ai-route-user-agent-override.md) | 中文

## Problem

按 `User-Agent` 白名单客户端的中转网关(agentrouter 一类,校验 `claude-cli/x.y.z (external, cli)`)在读取任何请求体之前就以 401 拒绝 harness 身份。`dsh-llm` 的归因头刻意设计为不可压制,且固定渲染 `product/version (+url)`,任何配置都声明不出白名单客户端字符串,这类网关完全够不着。

## Decision

在 `dsh-llm-pi-ai` 的请求头合并里,provider 路由自己声明的 `User-Agent`(大小写不敏感)胜过归因头;其余归因名仍归 Harness 拥有,未声明的路由照旧发送归因头。路由声明 `User-Agent` 就是在声明该网关的要求——与 `extraBody` 相同的"由部署方下断言"姿态。

## Alternatives considered

**白标 `AppIdentity`。** 归因模块本就接受自定义身份,但 `userAgent()` 渲染 `product/version (+url)`——RFC 9110 的 product 加注释形式,表达不出 `claude-cli/1.0.128 (external, cli)`,而且也没有路由级的身份传递接缝。

**经 pi-ai 的 `headers` 选项发送客户端串。** 本变更修正的合并顺序在 harness 适配器里;pi-ai 在适配器归因之前合并它自己的选项头,不改这里归因仍然获胜。

## Consequences

买到的:任何按客户端白名单的中转,仅凭 `settings.yaml` 就可达,要求写在配置处、看得见。

付出的:归因头不再保证出现在每个 provider 请求上——路由现在可以把 harness 扮成别的客户端。这是逐路由的显式部署声明而非静默默认:不声明该头的路由保持诚实身份,`headers` 的 JSDoc 写明了谁获胜。
