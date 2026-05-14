# Roadmap

## v0.1 — August 2026 [target]
- [x] System design + DESIGN.md
- [x] Algorithm specs (token-bucket, sliding-window-counter)
- [x] Multi-tenant data model
- [ ] Java service skeleton (Spring Boot 3, Java 21)
- [ ] Redis Lua scripts implemented + tested
- [ ] Postgres control plane + outbox
- [ ] Pub/sub cache invalidation
- [ ] Helm chart for k3d/EKS
- [ ] Integration tests via Testcontainers
- [ ] README usage section with curl examples
- [ ] Demo deployment on DOKS

## v0.2 — Q4 2026 [aspirational]
- [ ] Prometheus + OpenTelemetry instrumentation
- [ ] Chaos test suite (Redis kill, Postgres kill, pod kill)
- [ ] Performance benchmark vs `rate-limiter-flexible` baseline
- [ ] gRPC API surface

## v0.3 — 2027 [aspirational]
- [ ] Distributed sliding-window-log
- [ ] Multi-region cluster support
