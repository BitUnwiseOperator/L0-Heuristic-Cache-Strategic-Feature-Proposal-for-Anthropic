# Scaling Architecture: Edge Deployment & Bottleneck Mitigation

**Date:** 2025-11-04
**Status:** Critical - Addresses Scale & Latency at Production Load
**Problem:** Single Redis instance creates SPOF and latency bottleneck

---

## The Scaling Problem

### Current Architecture (Naive)
```
User (NYC) → L0L3Gateway (Virginia) → Redis (Virginia) → Response
              ↑ 20ms network latency ↑

User (Tokyo) → L0L3Gateway (Virginia) → Redis (Virginia) → Response
               ↑ 200ms network latency ↑
```

**Problems:**
1. ❌ **Single Point of Failure:** One Redis instance for all users
2. ❌ **Network Latency:** Geographic distance kills the <1ms L0 benefit
3. ❌ **Bottleneck:** All traffic funnels through one gateway
4. ❌ **No HA:** Redis crash = complete outage

**Reality Check:**
- **Our promise:** <1ms L0 cache lookup
- **Reality:** 20-200ms network latency to central Redis
- **Result:** We become a performance bottleneck, not an optimization

---

## The Solution: Edge Deployment + Cache Replication

### Correct Architecture (Edge-First)

```
                    ┌─────────────────────────────┐
                    │   Central HITL Write Path   │
                    │   (Single Source of Truth)  │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Redis Cluster (Primary)   │
                    │   - NYC Primary             │
                    │   - Tokyo Replica           │
                    │   - EU Replica              │
                    └──────────────┬──────────────┘
                                   │
                ┏━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━┓
                ┃        Read-Only Replication         ┃
                ┗━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┛
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
┌───────────────┐          ┌───────────────┐        ┌───────────────┐
│  Edge: NYC    │          │  Edge: Tokyo  │        │  Edge: EU     │
├───────────────┤          ├───────────────┤        ├───────────────┤
│ L0L3Gateway   │          │ L0L3Gateway   │        │ L0L3Gateway   │
│ Redis (local) │          │ Redis (local) │        │ Redis (local) │
│ <1ms latency  │          │ <1ms latency  │        │ <1ms latency  │
└───────┬───────┘          └───────┬───────┘        └───────┬───────┘
        │                          │                          │
        ▼                          ▼                          ▼
    User (NYC)               User (Tokyo)                User (EU)
    <1ms L0 hit              <1ms L0 hit                 <1ms L0 hit
```

**Key Principles:**
1. ✅ **Edge Deployment:** L0L3Gateway + Redis co-located at each region
2. ✅ **Read-Only Replicas:** Each edge has local Redis replica (eventual consistency)
3. ✅ **Central Write Path:** HITL UI writes to primary, replicates to edges
4. ✅ **<1ms Latency:** Local Redis lookup (no network hop)

---

## Deployment Patterns

### Pattern 1: Single-Region (Small Scale)

**Use Case:** Startup, single geography, <10K RPM

```
┌─────────────────────────────────────┐
│  Single Region (e.g., us-east-1)   │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────┐   ┌─────────────┐ │
│  │ L0L3Gateway │───│ Redis       │ │
│  │ (3 replicas)│   │ (1 primary) │ │
│  └─────────────┘   └─────────────┘ │
│                                     │
└─────────────────────────────────────┘
```

**Characteristics:**
- Simple deployment (single AZ or multi-AZ)
- Redis standalone or Sentinel (HA)
- Load balancer in front of gateway replicas
- Good for <1M requests/day

**Latency:**
- L0 hit: <2ms (local network)
- L3 miss: ~2s (LLM call)

---

### Pattern 2: Multi-Region (Global Scale)

**Use Case:** Global users, >100K RPM, <50ms p99 SLO

```
┌────────────────────────────────────────────────────────────┐
│                    Global Architecture                      │
└────────────────────────────────────────────────────────────┘

                     ┌──────────────────┐
                     │  Primary Region  │
                     │  (us-east-1)     │
                     ├──────────────────┤
                     │  HITL Web UI     │ ← ONLY Writer
                     │  Redis Primary   │
                     └────────┬─────────┘
                              │
               ┌──────────────┼──────────────┐
               │ Async Replication (Redis)   │
               └──────────────┬──────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐    ┌───────────────┐
│  Edge: NYC    │     │  Edge: Tokyo  │    │  Edge: EU     │
├───────────────┤     ├───────────────┤    ├───────────────┤
│ Gateway (3x)  │     │ Gateway (3x)  │    │ Gateway (3x)  │
│ Redis Replica │     │ Redis Replica │    │ Redis Replica │
│ Read-Only     │     │ Read-Only     │    │ Read-Only     │
└───────────────┘     └───────────────┘    └───────────────┘
```

**Replication Strategy:**
- **Write Path:** HITL UI → Primary Redis (us-east-1)
- **Read Path:** Users → Local edge gateway → Local Redis replica
- **Replication:** Redis async replication (primary → replicas)
- **Latency:** <5s replication lag (acceptable for cache updates)

**Characteristics:**
- Edge gateways in 3-5 regions
- Redis replica co-located with each edge
- Global load balancer (GeoDNS or Cloudflare)
- Good for 10M-1B requests/day

**Latency:**
- L0 hit: <1ms (local Redis)
- Replication lag: <5s (eventual consistency)

---

### Pattern 3: Kubernetes Edge (Extreme Scale)

**Use Case:** Massive scale, >1M RPM, <10ms p99 SLO

```
┌────────────────────────────────────────────────────────────┐
│              Kubernetes Multi-Cluster Edge                  │
└────────────────────────────────────────────────────────────┘

Each Region has:
  ┌─────────────────────────────────────┐
  │  K8s Cluster (e.g., us-east-1)     │
  ├─────────────────────────────────────┤
  │  ┌─────────────────────────────┐   │
  │  │  L0L3Gateway Pods (20x)     │   │  ← Auto-scaled
  │  │  - HPA: 10-100 pods         │   │
  │  │  - CPU: 0.5-2 cores/pod     │   │
  │  └─────────────────────────────┘   │
  │                                     │
  │  ┌─────────────────────────────┐   │
  │  │  Redis Cluster (Sidecar)    │   │  ← Co-located
  │  │  - StatefulSet (3 nodes)    │   │
  │  │  - Local SSD (low latency)  │   │
  │  └─────────────────────────────┘   │
  │                                     │
  │  ┌─────────────────────────────┐   │
  │  │  Service Mesh (Istio)       │   │  ← Observability
  │  │  - p99 latency tracking     │   │
  │  │  - Circuit breaking         │   │
  │  └─────────────────────────────┘   │
  └─────────────────────────────────────┘
```

**Deployment:**
```yaml
# gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l0l3-gateway
spec:
  replicas: 20
  template:
    spec:
      containers:
      - name: gateway
        image: constraint-cache:v1.0
        env:
        - name: REDIS_HOST
          value: "localhost"  # Sidecar
        - name: REDIS_PORT
          value: "6379"
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi

      - name: redis-sidecar
        image: redis:7-alpine
        volumeMounts:
        - name: redis-data
          mountPath: /data
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
```

**Characteristics:**
- Gateway pods auto-scale (HPA on CPU/latency)
- Redis sidecar pattern (co-located, <1ms)
- Service mesh for observability
- Good for 100M-10B requests/day

**Latency:**
- L0 hit: <1ms (sidecar Redis)
- Pod-to-pod: <1ms (K8s network)
- Auto-scaling: <30s (scale out on load spike)

---

## Bottleneck Analysis

### 1. Redis Throughput Bottleneck

**Problem:** Redis single-threaded, ~100K ops/sec max

**Solution:**
```
Option 1: Redis Cluster (Sharding)
  - Shard by intent key (e.g., hash("cancel_order") % 16)
  - 16 shards = 1.6M ops/sec
  - Co-locate shards with gateway pods

Option 2: Redis on GPU (RediSearch)
  - 10x throughput on GPU-accelerated Redis
  - Good for semantic search (L1/L2 layers)

Option 3: In-Process Cache (Caffeine)
  - L0 cache in gateway process memory (JVM)
  - 10M ops/sec per gateway
  - TTL-based sync with Redis (eventual consistency)
```

**Recommended:** In-Process Cache (L0.5) for extreme scale

```python
class L0L3Gateway:
    def __init__(self, ...):
        self.l0_redis = Adaptive_L0_Cache(redis_client)
        self.l0_memory = InMemoryCache(ttl=60)  # L0.5 (hot cache)

    def chat(self, prompt):
        # L0.5: In-memory hot cache (10M ops/sec)
        result = self.l0_memory.get(prompt)
        if result:
            return result

        # L0: Redis cache (100K ops/sec)
        result = self.l0_redis.get(prompt)
        if result:
            self.l0_memory.set(prompt, result)  # Populate L0.5
            return result

        # L1, L2, L3...
```

**Result:**
- 99% of traffic served from in-memory (L0.5)
- 1% of traffic hits Redis (L0)
- No Redis bottleneck

---

### 2. Network Latency Bottleneck

**Problem:** Cross-region network latency kills <1ms SLO

**Solution:** Edge deployment (already covered)

**Latency Budget:**
```
Target: <1ms L0 cache hit

Breakdown:
  - Gateway processing:     0.2ms  (scrubbing + normalization)
  - Redis local lookup:     0.5ms  (co-located, same pod/node)
  - Network overhead:       0.2ms  (localhost or pod network)
  - Response serialization: 0.1ms
  ────────────────────────────────
  Total:                    1.0ms  ✅ Meets SLO
```

**If Redis is remote:**
```
Breakdown:
  - Gateway processing:     0.2ms
  - Network to Redis:      20.0ms  ❌ Kills latency budget
  - Redis lookup:           0.5ms
  - Network back:          20.0ms
  ────────────────────────────────
  Total:                   40.7ms  ❌ 40x over budget
```

**Conclusion:** Redis MUST be co-located (same pod, same node, or same AZ)

---

### 3. Gateway Processing Bottleneck

**Problem:** IPiiScrubber (Presidio) adds 200ms latency

**Solution:** Already addressed (NoOpScrubber by default)

**Latency Options:**
```
NoOpScrubber:           <0.1ms  ✅ Zero overhead
Custom Heuristics:      <1ms    ✅ Fast regex
Presidio (naive):      ~200ms   ❌ Too slow
Presidio (optimized):   ~20ms   ⚠️ Acceptable but expensive
```

**Scaling Strategy:**
1. **Default:** NoOpScrubber (users handle PII upstream)
2. **Opt-in:** Custom heuristics (domain-specific, <1ms)
3. **Last resort:** Optimized Presidio (only if required)

---

## Cache Consistency Model

### Write Path (Strong Consistency)

```
HITL Approval → Primary Redis (us-east-1) → Replication → Edge Replicas
                ↑ Single Writer              ↑ Async       ↑ <5s lag
```

**Guarantees:**
- ✅ Single writer (HITL UI only)
- ✅ All edges eventually see updates (<5s)
- ✅ No write conflicts (read-only edges)

### Read Path (Eventual Consistency)

```
User Query → Edge Gateway → Local Redis Replica → Response
             ↑ Read-only     ↑ May be stale (<5s)
```

**Guarantees:**
- ✅ <1ms latency (local read)
- ⚠️ May serve stale cache (<5s old)
- ✅ No data loss (primary is source of truth)

**Is 5s lag acceptable?**

**YES** for L0 cache because:
1. Cache entries are **generic instructions** (not real-time data)
2. Updates are **rare** (HITL approval happens hours/days apart)
3. Serving stale cache is **better than L3 LLM call** (cost/latency)

**Example:**
```
T=0:     HITL approves "cancel_order" → "To cancel, visit..."
T=5s:    Tokyo edge sees update (replicated)
T=0-5s:  Tokyo users get cache miss → L3 call (acceptable)
T=5s+:   Tokyo users get cache hit (<1ms)
```

**Worst case:** 5 seconds of cache misses for 1 intent in 1 region = **negligible impact**

---

## High Availability Architecture

### Single Region HA (3 Nines: 99.9%)

```
┌─────────────────────────────────────────┐
│  Region: us-east-1                      │
├─────────────────────────────────────────┤
│  ┌───────────────────────────────────┐  │
│  │  ALB (Load Balancer)              │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│      ┌───────┼───────┐                  │
│      ▼       ▼       ▼                  │
│  ┌─────┐ ┌─────┐ ┌─────┐               │
│  │ GW1 │ │ GW2 │ │ GW3 │ ← 3 replicas  │
│  └──┬──┘ └──┬──┘ └──┬──┘               │
│     │       │       │                   │
│     └───────┼───────┘                   │
│             ▼                           │
│  ┌─────────────────────────────────┐   │
│  │  Redis Sentinel (3 nodes)       │   │
│  │  - 1 Primary + 2 Replicas       │   │
│  │  - Auto-failover (<30s)         │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Failure Modes:**
- Gateway pod crash → ALB routes to healthy pods (no downtime)
- Redis primary crash → Sentinel promotes replica (<30s downtime)
- AZ failure → Multi-AZ deployment maintains 2/3 quorum

**SLO:** 99.9% uptime (~8 hours downtime/year)

---

### Multi-Region HA (5 Nines: 99.999%)

```
┌─────────────────────────────────────────────────────────┐
│  Global DNS (Route53 or Cloudflare)                    │
│  - Health checks on all regions                         │
│  - Failover: us-east-1 down → route to eu-west-1       │
└─────────────────┬───────────────────────────────────────┘
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
  ┌────────┐  ┌────────┐  ┌────────┐
  │ us-e-1 │  │ eu-w-1 │  │ ap-ne-1│
  │ Active │  │ Active │  │ Active │
  └────────┘  └────────┘  └────────┘
```

**Failure Modes:**
- Region failure → GeoDNS routes to next-closest region
- Gateway failure → Regional load balancer handles
- Redis failure → Regional Sentinel handles

**SLO:** 99.999% uptime (~5 minutes downtime/year)

---

## Performance Benchmarks (Expected)

### Single Gateway Instance
```
Configuration: 4 vCPU, 8GB RAM, co-located Redis

L0 Cache Hits:
  - Throughput:  50,000 req/sec
  - Latency p50: 0.8ms
  - Latency p99: 2.0ms

L3 Cache Misses:
  - Throughput:  100 req/sec (LLM bottleneck)
  - Latency p50: 2000ms
  - Latency p99: 5000ms
```

### Edge Deployment (3 Regions, 20 Gateways Each)
```
Configuration: 60 gateways total, Redis replicas

Global Throughput:
  - L0 Hits:  3,000,000 req/sec
  - L3 Calls:    30,000 req/sec

Regional Latency (p99):
  - NYC:   <2ms
  - Tokyo: <2ms
  - EU:    <2ms
```

### With In-Memory L0.5 Cache
```
Configuration: 60 gateways, in-process cache

Global Throughput:
  - L0.5 Hits:  30,000,000 req/sec  (10x improvement)
  - Redis Hits:    300,000 req/sec  (1% miss rate)
```

---

## Deployment Recommendations

### Startup (< 1M req/day)
```
Single Region Deployment:
  - 3 gateway replicas (2 vCPU each)
  - 1 Redis instance (Sentinel HA)
  - Cost: ~$200/month
  - SLO: 99.9% uptime, <5ms p99
```

### Growth (1M - 100M req/day)
```
Multi-Region Deployment:
  - 3 regions (us, eu, asia)
  - 10 gateways per region
  - Redis replication (primary + 3 replicas)
  - Cost: ~$2K/month
  - SLO: 99.95% uptime, <2ms p99
```

### Enterprise (> 100M req/day)
```
Kubernetes Multi-Cluster:
  - 5+ regions
  - 20-100 pods per region (auto-scaled)
  - Redis clusters (sharded)
  - In-memory L0.5 cache
  - Cost: ~$20K/month
  - SLO: 99.999% uptime, <1ms p99
```

---

## Open Questions & Next Steps

### 1. Cache Invalidation at Scale

**Question:** How do we invalidate a cache entry across all edges?

**Options:**
- **Option A:** Pub/Sub (Redis) - Fast but requires connection pooling
- **Option B:** TTL-based (60s) - Simple but 60s stale window
- **Option C:** Version tags - Complex but precise

**Recommendation:** Start with TTL-based (v1.0), add Pub/Sub (v2.0)

### 2. Observability at Scale

**Question:** How do we track p99 latency across 60+ gateways?

**Required Metrics:**
- `l0_cache_hit_latency_p99` (by region)
- `l0_cache_miss_count` (by intent)
- `redis_replication_lag` (by edge)

**Recommendation:** Prometheus + Grafana (standard observability stack)

### 3. Cost Model

**Question:** What's the TCO at 1B requests/day?

**Estimated Costs:**
```
Compute (60 gateways):      $5K/month
Redis (15 nodes):           $3K/month
Network (inter-region):     $2K/month
Observability:              $1K/month
────────────────────────────────────────
Total:                     $11K/month

Cost per 1M requests: $0.33
vs LLM cost (no cache): $10-100 per 1M requests

ROI: 30-300x cost reduction
```

---

## Summary: Scaling Architecture Decisions

| Component | Single Region | Multi-Region | K8s Edge |
|-----------|--------------|--------------|----------|
| **Gateway Replicas** | 3 | 30 (10 per region) | 100+ (auto-scaled) |
| **Redis Topology** | Sentinel | Primary + 3 replicas | Cluster (sharded) |
| **Deployment** | Docker Compose | ECS/VM | Kubernetes |
| **Latency p99** | <5ms | <2ms | <1ms |
| **Throughput** | 50K req/sec | 3M req/sec | 30M req/sec |
| **Cost** | $200/mo | $2K/mo | $20K/mo |
| **HA SLO** | 99.9% | 99.95% | 99.999% |

**Key Principle:** Redis MUST be co-located with gateway (same pod/node/AZ) to achieve <1ms latency

---

## Action Items for v1.0

### Must Have (Blocking v1.0)
- [ ] Document deployment patterns (single-region, multi-region)
- [ ] Add latency tracking to L0L3Gateway (.dashboard() shows p99)
- [ ] Example: Docker Compose with co-located Redis
- [ ] Example: K8s deployment YAML

### Should Have (v1.5)
- [ ] Redis replication support (primary + replicas)
- [ ] In-memory L0.5 cache (Caffeine/Redis local cache)
- [ ] Observability integration (Prometheus metrics)

### Nice to Have (v2.0)
- [ ] Cache invalidation Pub/Sub
- [ ] Auto-scaling examples (HPA)
- [ ] Multi-region benchmark results

---

## Conclusion

**The Answer to Your Question:**

> "How does our system scale? 1 large shared cache across a location?"

**No.** That would create a bottleneck.

**Correct Architecture:**
1. **Edge Deployment:** Gateway + Redis at each region (co-located)
2. **Read-Only Replicas:** Each edge has local Redis (eventual consistency)
3. **Central Write Path:** HITL UI writes to primary, replicates to edges
4. **In-Memory L0.5:** For extreme scale (optional)

**Result:**
- ✅ <1ms latency (local Redis lookup)
- ✅ No bottleneck (each edge is independent)
- ✅ Linear scaling (add more edges)
- ✅ High availability (multi-region failover)

**We don't become a point of weakness—we become a distributed edge layer.**
