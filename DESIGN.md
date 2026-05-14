# FluxLimiter Design Document

> Status: Design-complete, implementation in progress. v0.1 target August 2026.

## Goals

1. **Multi-tenant**: distinct rate-limit policies per tenant, configurable at runtime without service restart.
2. **Algorithm-pluggable**: token-bucket (burst-tolerant) and sliding-window-counter (strict SLO) behind a common decision API.
3. **Atomic decisions**: no read-decide-write race window between concurrent requests for the same key.
4. **Graceful degradation**: configurable fail-open vs. fail-closed when Redis is unreachable.
5. **Kubernetes-native**: ships as a Deployment + Service + Helm chart, scales horizontally.

## Non-goals (v0.1)

- Cross-region rate limiting (single Redis cluster scope).
- Quota carry-over across windows.
- ML-based abuse detection.

## Architecture

### Components

- **FluxLimiter service** (Java 21, Spring Boot 3) — stateless, horizontally scalable. Exposes REST `/v1/check` and gRPC.
- **Redis 7** — limiter state (bucket levels, window counts) accessed via atomic `EVAL` Lua scripts.
- **Postgres 16** — rule control plane: per-tenant policy definitions, audit log of policy changes.
- **Outbox + pub/sub** — Postgres outbox table → Redis pub/sub fan-out → in-process LRU cache invalidation across all FluxLimiter pods.

### Request flow

1. Client calls `POST /v1/check` with `{tenant_id, resource, key}`.
2. Service resolves the policy for `(tenant_id, resource)` from in-memory cache (Caffeine, ~1ms).
3. Service issues `EVAL` to Redis with the appropriate Lua script (token-bucket or sliding-window).
4. Lua script atomically reads counter state, decides allow/deny, writes new state, returns decision.
5. Service returns `{allowed: true|false, retry_after_ms, remaining}`.

### Algorithm: Token Bucket (Lua)

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity, ARGV[2] = refill_rate_per_sec, ARGV[3] = now_ms, ARGV[4] = cost
local data = redis.call('HMGET', KEYS[1], 'tokens', 'last_refill_ms')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local last_refill = tonumber(data[2]) or tonumber(ARGV[3])
local elapsed_sec = (tonumber(ARGV[3]) - last_refill) / 1000
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed_sec * tonumber(ARGV[2]))
local cost = tonumber(ARGV[4])
local allowed = 0
if tokens >= cost then
  tokens = tokens - cost
  allowed = 1
end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'last_refill_ms', ARGV[3])
redis.call('PEXPIRE', KEYS[1], 3600000)
return {allowed, tokens}
```

**Why Lua here:** without it, the read–decide–write sequence has a race window. Two concurrent requests can both read `tokens=1`, both decide allow, both write `tokens=0` — a 2× burst leak. Wrapping in `EVAL` makes Redis serialize the whole script on its single-threaded executor.

### Algorithm: Sliding-Window-Counter

Maintains the current window count and the previous window count, weights the previous window by how much of it overlaps the current sliding window. Approximates true sliding-window-log without storing every request timestamp. Documented error: up to ~50% over the limit in adversarial bursts at the window boundary — acceptable for general API throttling, not for billing.

### Multi-tenant data model (Postgres)

```sql
CREATE TABLE tenants (
  id           UUID PRIMARY KEY,
  name         TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE rate_limit_policies (
  id              UUID PRIMARY KEY,
  tenant_id       UUID NOT NULL REFERENCES tenants(id),
  resource        TEXT NOT NULL,
  algorithm       TEXT NOT NULL CHECK (algorithm IN ('token_bucket', 'sliding_window')),
  capacity        INT NOT NULL,
  refill_per_sec  NUMERIC NOT NULL,
  fail_mode       TEXT NOT NULL CHECK (fail_mode IN ('open', 'closed')),
  version         INT NOT NULL DEFAULT 1,
  UNIQUE (tenant_id, resource)
);

CREATE TABLE policy_outbox (
  id            BIGSERIAL PRIMARY KEY,
  policy_id     UUID NOT NULL,
  event_type    TEXT NOT NULL,
  payload       JSONB NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at  TIMESTAMPTZ
);
```

Redis keys are namespaced as `flux:{tenant_id}:{resource}:{key}` so `{tenant_id}` becomes the Redis Cluster hash tag — all of a tenant's keys live in the same slot, enabling future per-tenant Lua transactions.

### Fail mode

When Redis is unreachable, Resilience4j circuit-breaker opens. Behavior is per-policy:
- `fail_mode = 'open'`: allow all requests, increment a `flux_fallback_total` counter, page on >30s sustained.
- `fail_mode = 'closed'`: deny all requests with `503 Service Unavailable`, page immediately.

Trade-off: open mode trusts callers during outage and risks abuse leak; closed mode protects backend but causes user-visible failure. Defaults to open for read-heavy resources, closed for payment/auth.

## Trade-offs and known limits

| Decision | Why | Cost |
|---|---|---|
| Lua over `INCR + EXPIRE` | Atomic; closes race | Slightly more complex script management |
| Token bucket as default | Burst-tolerant, fits API traffic | Up to 2× burst at refill |
| Sliding-window-counter (not log) | Cheap memory | Up to ~50% error at boundary |
| In-memory policy cache | Sub-ms lookup | Cache invalidation via pub/sub |
| Async replication (Redis) | Throughput | Up to replication lag of double-counting on failover |
| Per-tenant Redis hash tags | Future-proofs cluster ops | Hot tenants can hot-spot a slot |

## Out of scope (deferred to v0.2+)

- Distributed sliding-window-log (true precision)
- Multi-region active-active
- Quota leasing for offline use
- ML-based anomaly detection

## References

- Stripe rate-limiter blog (2017)
- Cloudflare's sliding-window-counter post
- "How we built rate limiting capable of scaling to millions of domains" — Figma
- DDIA Ch 5, Ch 9 (Kleppmann)
