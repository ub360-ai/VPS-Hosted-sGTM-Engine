# sGTM Scaling Upgrade Plan — Million Traffic Resilience

**Target URL:** `https://sgtm.updates24-7.swiftloopmarketing.com/`
**Orchestration:** Coolify (Docker + Traefik)
**Peak Traffic:** ~500–2000 req/s
**Problem:** "No server available" — container crashes under load, single point of failure, no queue, in-memory state lost on restart.

---

## Target Architecture

```
Cloudflare Edge (orange-cloud)
    │
    ▼
sgtm.updates24-7.swiftloopmarketing.com
    │
    ▼
VPS: Traefik (Rate Limit / Retry / Circuit Breaker)
    │
    ├── gtm-cluster-replica-1 ──┐
    ├── gtm-cluster-replica-2 ──┼── Redis (dedup + buffer)
    ├── gtm-cluster-replica-3 ──┘
    │
    └── gtm-preview (single, debug only)
```

---

## Root Causes of "No Server Available"

| # | Root Cause | Why It Happens |
|---|---|---|
| 1 | **OOM Kill** | No `mem_limit` — container consumes all host RAM → Docker kills it |
| 2 | **Single Replica** | One container crashes → zero backends → Traefik returns 502/503 |
| 3 | **No Health Check** | Container starts but app is unresponsive — Docker doesn't know |
| 4 | **No Backpressure** | Traffic spike overwhelms Go runtime → requests pile up → Go GC thrashing |
| 5 | **In-Memory State Loss** | Dedup keys lost on restart → duplicate events + re-processing storm |
| 6 | **No Monitoring** | Crashes discovered by users, not by ops |

---

## Phase Plan

### Phase 1 — Container Hardening
**File:** `docker-compose.yml`

Add to `tagging` service:
- `restart: on-failure:5` (prevents infinite restart loops)
- `mem_limit: 2560m`, `mem_reservation: 1024m` (OOM guard)
- `cpus: '4'` (CPU ceiling)
- `GOMAXPROCS=4`, `GOMEMLIMIT=2048MiB` (Go runtime guard)
- `healthcheck` to `/healthz` with retries
- `stop_grace_period: 30s` (drain connections)
- Traefik labels: `sticky.cookie`, `healthcheck.path`

Add to `preview` service:
- Same health check and resource limits

### Phase 2 — Redis (Dedup + Buffer)
**File:** `docker-compose.yml`

New `redis` service:
- `redis:7-alpine`
- `--appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru`
- Named volume `redis_data`
- Health check via `redis-cli ping`

Purpose:
- Dedup keys survive container restarts (SETNX with 24h TTL)
- Future: async queue buffer for backpressure

### Phase 3 — Replica Scaling
**File:** `docker-compose.yml` + Coolify Advanced Settings

- `deploy.replicas: 3` in docker-compose.yml on tagging service
- **Critical Coolify step:** In Coolify Service > Advanced, DISABLE:
  - "Consistent Container Names"
  - "Custom Container Name" (leave blank)
  - These inject `container_name` into the compose file, which conflicts with `deploy.replicas`
- Traefik sticky sessions ensure requests from same client hit same replica

### Phase 4 — Traefik Resilience
**File:** `traefik/dynamic.yml`

Three middlewares:
1. **Rate Limit:** 2000 avg, 500 burst (drops excess before hitting backends)
2. **Retry:** 3 attempts, 100ms interval (recovers from transient errors)
3. **Circuit Breaker:** Opens when 10%+ of backends error, 10s check period, 30s fallback

### Phase 5 — Environment Documentation
**File:** `.env.example`

All required env vars documented with defaults and descriptions.

### Phase 6 — Monitoring
**File:** `monitoring/docker-compose.monitor.yml`

Uptime Kuma stack:
- Pings `/healthz` every 30s
- Alerts via Telegram/email on failure
- Persistent volume for config

### Phase 7 — Cloudflare (External, No Code)
- Enable orange-cloud proxy on the DNS record
- Adds DDoS protection, edge caching, and SSL termination

---

## Files Modified / Created

| File | Action | Phase |
|---|---|---|
| `docker-compose.yml` | **Modify** | 1, 2, 3 |
| `traefik/dynamic.yml` | **Create** | 4 |
| `.env.example` | **Create** | 5 |
| `monitoring/docker-compose.monitor.yml` | **Create** | 6 |

---

## Post-Deployment Verification

1. `curl -s https://sgtm.updates24-7.swiftloopmarketing.com/healthz` → 200 OK
2. `docker stats` → memory stays under 2GB per tagging container
3. `docker ps` → 3 tagging replicas running, Redis running
4. Emulate bulk requests and confirm no 502/503 from Traefik
5. Restart one replica — confirm traffic shifts without errors
