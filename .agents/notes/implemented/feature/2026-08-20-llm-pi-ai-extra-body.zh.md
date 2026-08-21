# Agent Note: pi-ai 路由 `extraBody` 请求字段逃生口

Status: implemented

[English](2026-08-20-llm-pi-ai-extra-body.md) | 中文

## Problem

企业级 OpenAI 兼容网关可能要求 compat 开关没有覆盖的请求体字段——常见的是用于按用户计费的 `user`。pi-ai 自己拼装整个请求体且没有透传能力,这类网关后面的部署不改依赖源码就加不上字段。

## Decision

`PiAiProviderProfile.extraBody`(自由形态的 `Record<string, unknown>`)通过 pi-ai 的 `onPayload` 钩子合并进该路由每个外发请求体,钩子在协议拼装完请求体之后触发。部署方拥有的键在同名冲突时获胜,因此契约对两个方向都直言:覆盖协议自有键可能损坏请求,端点不接受的键会让该请求失败。缺省保持缺省——没有配置 `extraBody` 的路由不挂 `onPayload`,行为与之前完全一致。schema 对每个键接受任意 JSON 值(`z.dict(z.any())`),因为定义这些字段含义的是端点,不是本包。

提示缓存标识不需要改代码:`cacheRetention: long` 加 `compat.supportsLongCacheRetention: true` 即让 pi-ai 发送 `prompt_cache_key`(Harness 会话 id,同一会话的请求间稳定)和 `prompt_cache_retention: "24h"`。两个字段均已通过本地截获服务与真实网关在线上验证。

## Alternatives considered

**专设 `requestUser` 字段。** 更窄、自解释,但每个网关特例加一个类型化字段会让这个循环永不收敛;自由形态的字典用一个接缝回应了整个家族(`user`、厂商开关、路由提示)。

**给 pi-ai 打补丁。** 请求体拼装归依赖所有,`pnpm patch` 可行,但依赖已经发布了 `onPayload` 钩子,打补丁等于把 fork 钉死在每个上游版本上。

**会话亲和请求头。** pi-ai 的 `sendSessionAffinityHeaders` 把身份放在 HTTP 头而不是请求体字段里,且 dsh 的 compat 面没有开放这些开关;这里的网关读取的是请求体字段。

## Consequences

买到的:任何部署在 `settings.yaml` 里声明其网关要求的请求体字段,字段就随该路由的每个请求外出,不需要 fork 依赖,也不需要超出这个钩子的适配器改动。

付出的:`extraBody` 是类型系统无法校验的对端点的声明。写错键名要到网关那里才失败,而不是加载时;与协议自有键同名会静默损坏该请求。JSDoc 在字段配置处写明了这两个风险,自由形态的值也不受任何 compat 漂移门的保护——将来 pi-ai 正式命名的字段应迁移为类型化开关,让 `extraBody` 留空。
