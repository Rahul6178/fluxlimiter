README.md

# FluxLimiter

[![Status: WIP](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> 🚧 **Status: Design phase, active development.** v0.1 milestone targeted **August 2026**.
> See [DESIGN.md](./DESIGN.md) and [ROADMAP.md](./ROADMAP.md).

A Kubernetes-native multi-tenant distributed rate limiter built in Java, Redis, and Lua. Token-bucket and sliding-window-counter algorithms, per-tenant policy isolation via a Postgres rule control plane, configurable fail-open/fail-closed degradation.

## Why

Existing rate limiters either (a) live inside a single application's process and don't share state across replicas, or (b) ship as opinionated SaaS where you can't tune the algorithm. FluxLimiter targets the gap: a small, self-hostable, Kubernetes-native rate-limiter service that exposes both algorithms behind a uniform API, with per-tenant policy isolation suitable for B2B platforms.

## Status

| Component                              | Status            |
|----------------------------------------|-------------------|
| System design + architecture           | ✅ Complete       |
| Token-bucket Lua algorithm spec        | ✅ Complete       |
| Sliding-window-counter algorithm spec  | ✅ Complete       |
| Multi-tenant policy data model         | ✅ Complete       |
| Java service skeleton                  | 🚧 Planned (Jul)  |
| Redis Lua scripts                      | 🚧 Planned (Jul)  |
| Postgres control plane                 | 🚧 Planned (Jul)  |
| Helm chart                             | 🚧 Planned (Jul)  |
| Prometheus metrics                     | ⏳ Planned (Aug)  |
| End-to-end demo deploy                 | ⏳ Planned (Aug)  |

## Architecture (preview)

```
                      ┌─────────────────┐
                      │ Postgres        │
                      │ (rule control)  │
                      └────────┬────────┘
                               │ outbox + pub/sub
                               ▼
   client ──HTTP──▶ ┌──────────────────┐ ──EVAL──▶ ┌────────┐
                    │ FluxLimiter      │           │ Redis  │
                    │ (Java/Spring)    │ ◀──────── │ (Lua)  │
                    └──────────────────┘           └────────┘
                               │
                               ▼
                      Prometheus / OTEL
```

See [DESIGN.md](./DESIGN.md) for the full architecture rationale, algorithm specs, and trade-off analysis.

## License MIT 


