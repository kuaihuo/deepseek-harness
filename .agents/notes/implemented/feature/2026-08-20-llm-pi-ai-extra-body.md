# Agent Note: pi-ai route `extraBody` request-field escape

Status: implemented

English | [中文](2026-08-20-llm-pi-ai-extra-body.zh.md)

## Problem

Corporate OpenAI-compatible gateways can require request-body fields no compat switch names — `user` for per-user accounting is the common one. pi-ai assembles the whole request body and offers no passthrough, so a deployment behind such a gateway could not add the field without patching the dependency.

## Decision

`PiAiProviderProfile.extraBody` (a free-form `Record<string, unknown>`) merges into every outgoing request body for that route's models through pi-ai's `onPayload` hook, which fires after the protocol assembles the body. Deployment-owned keys win same-name collisions, so the contract is honest about both directions: a key the protocol owns can corrupt the request, and a key the endpoint rejects fails that request. Absent stays absent — routes without `extraBody` take no `onPayload` and behave exactly as before. The schema accepts any JSON value per key (`z.dict(z.any())`) because the endpoint, not this package, defines what the fields mean.

Prompt-cache identification needed no code: `cacheRetention: long` plus `compat.supportsLongCacheRetention: true` makes pi-ai send `prompt_cache_key` (the Harness session id, stable across a session's requests) and `prompt_cache_retention: "24h"`. Both fields were verified on the wire against a local request sink and a live gateway.

## Alternatives considered

**A dedicated `requestUser` field.** Narrower and self-documenting, but one typed field per gateway quirk repeats the cycle this change closes; the free-form dict answers the whole family (`user`, vendor toggles, routing hints) with one seam.

**Patching pi-ai.** The library owns the body assembly, so a `pnpm patch` would work, but it pins the fork against every upstream release for a hook the dependency already publishes.

**Session-affinity headers.** pi-ai's `sendSessionAffinityHeaders` sends identity as HTTP headers, not body fields, and the dsh compat surface does not expose those switches; the gateway in question reads the body fields.

## Consequences

What it bought: any deployment states its gateway's required body fields in `settings.yaml` and they travel with every request on that route, with no dependency fork and no adapter fork beyond this hook.

What it cost: `extraBody` is a claim about an endpoint the type system cannot check. A mis-keyed entry fails at the gateway rather than at load, and a key that collides with one the protocol owns corrupts that request silently. The JSDoc states both hazards where the field is configured, and the free-form values stay outside every compat gate's drift protection — fields a future pi-ai release names properly should migrate to typed switches and leave `extraBody` empty.
