# Agent Note: route-declared `User-Agent` overrides Harness attribution

Status: implemented

English | [中文](2026-08-25-llm-pi-ai-route-user-agent-override.zh.md)

## Problem

Relay gateways that whitelist clients by `User-Agent` (agentrouter-style `claude-cli/x.y.z (external, cli)` checks) refuse the harness identity with 401 before any body is read. `dsh-llm` attribution is deliberately unsuppressible and always renders `product/version (+url)`, so no configuration could state a whitelisted client string and those gateways were unreachable.

## Decision

In `dsh-llm-pi-ai`'s header merge, a provider route that declares its own `User-Agent` (case-insensitive) wins over the attribution header; every other attribution name stays Harness-owned, and a route that declares none sends attribution exactly as before. A route stating a `User-Agent` is declaring that gateway's requirement, the same claim-a-deployment-makes posture as `extraBody`.

## Alternatives considered

**A white-label `AppIdentity`.** The attribution module already accepts one, but `userAgent()` renders `product/version (+url)` — an RFC 9110 product-plus-comment form that cannot express `claude-cli/1.0.128 (external, cli)`, and no route-level seam passes an identity anyway.

**Sending the client string via pi-ai `headers` options.** The merge order this change fixes lives in the harness adapter; pi-ai merges its option headers before the adapter's attribution, so attribution still won without this change.

## Consequences

What it bought: any client-whitelisting relay becomes reachable from `settings.yaml` alone, with the requirement visible where it is configured.

What it cost: attribution is no longer guaranteed on every provider request — a route can now present the harness as another client. That is a deliberate, per-route deployment claim rather than a silent default: routes without the header keep the honest identity, and the JSDoc on `headers` states who wins.
