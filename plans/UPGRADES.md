# sGTM Scaling Upgrade Plan — Million Traffic Resilience

**Target URL:** `https://sgtm.updates24-7.swiftloopmarketing.com/`
**Orchestration:** Coolify (Docker + Traefik)
**Peak Traffic:** ~500–2000 req/s
**Problem:** "No server available" — container crashes under load, single point of failure, no queue, in-memory state lost on restart.

---

## Target Architecture

```
Internet (Direct, no proxy)
    │
    ▼
sgtm.updates24-7.swiftloopmarketing.com
    │
    ▼
VPS: Traefik (Rate Limit / Retry / Security Headers)
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
**File:** `docker-compose.yml`

- **Approach:** 3 separate services (`tagging`, `tagging-2`, `tagging-3`) via YAML anchors
- **Why not `deploy.replicas`:** Coolify injects `container_name` at the infrastructure level, permanently conflicting with `deploy.replicas`
- All 3 services share identical Traefik labels → they register as one backend pool (`gtm-tagging`)
- Traefik sticky sessions ensure requests from same client hit same replica

### Phase 4 — Traefik Resilience
**File:** Docker labels in `docker-compose.yml` (no separate file)

All Traefik configuration lives in the `x-tagging` anchor labels:
1. **Router:** `gtm-tagging` — HTTPS, Let's Encrypt cert, host-rule-based
2. **Middlewares** (defined via labels, not file): Rate Limit (2000/500), Retry (3x), Security Headers
3. **Service:** all 3 replicas register as servers under `gtm-tagging`, health check on `/healthz`, sticky sessions
4. **Why no `traefik/dynamic.yml`:** File-based config conflicts with Docker-discovered services in Traefik, causing "No available server". Labels-only avoids the merge conflict.

### Phase 5 — Environment Documentation
**File:** `.env.example`

All required env vars documented with defaults and descriptions.

### Phase 6 — Monitoring
**File:** `monitoring/docker-compose.monitor.yml`

Two-layer monitoring:
- **Uptime Kuma** (`:3001`): Pings `/healthz` every 30s, alerts via Telegram/email on failure. Monitors the public URL — Traefik routes to any healthy backend so an alert only fires if ALL replicas are down.
- **cAdvisor** (`:8088`): Per-container resource metrics (CPU, memory, network per replica). Use to spot a single misbehaving replica before it takes down the whole pool.
- Persistent volume for Kuma config

---

## Why No Cloudflare Proxy
Cloudflare proxy (orange cloud) strips the original client IP address. For server-side tracking, the real IP is critical for:
- Geo-location attribution
- IP-based deduplication signals
- Accurate `ip_override` in Meta CAPI events

Traffic flows directly to the VPS. Coolify/Traefik handles SSL via Let's Encrypt HTTP-01 challenge, which works without Cloudflare. SSL renewal is automatic.

---

## Files Modified / Created

| File | Action | Phase |
|---|---|---|---|
| `docker-compose.yml` | **Modify** | 1, 2, 3, 4 |
| `.env.example` | **Create** | 5 |
| `monitoring/docker-compose.monitor.yml` | **Create** | 6 |

---

## Post-Deployment Verification

1. `curl -s https://sgtm.updates24-7.swiftloopmarketing.com/healthz` → 200 OK
2. `docker stats` → memory stays under 2GB per tagging container
3. `docker ps` → 3 tagging replicas running, Redis running
4. Emulate bulk requests and confirm no 502/503 from Traefik
5. Restart one replica — confirm traffic shifts without errors
