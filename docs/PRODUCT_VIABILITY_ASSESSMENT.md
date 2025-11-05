# Product Viability Assessment: L0 Heuristic Cache

**Date:** 2025-11-04
**Status:** 🔴 CRITICAL REASSESSMENT REQUIRED
**Question:** Is this product still feasible given the operational complexity we've uncovered?

---

## Executive Summary

**THE PROBLEM:**
We started designing a "simple pluggable cache" and ended up with **enterprise-grade security infrastructure** requiring MFA, HITL approvers, Redis Sentinel, circuit breakers, audit logging, and 24/7 ops support.

**KEY QUESTION:** Is the juice worth the squeeze?

**ANSWER:** **It depends on the distribution model.** The product is viable BUT requires a fundamental decision on packaging:

1. ✅ **SaaS (Anthropic/OpenAI hosts)** → HIGHLY VIABLE ($5-20M ARR potential)
2. ⚠️ **Self-Hosted Enterprise** → VIABLE but only for large orgs (>1000 employees)
3. ❌ **Open-Source Self-Hosted** → NOT VIABLE (too complex for small teams)

---

## Part 1: Where We Started vs Where We Are

### Where We Started (Week 1)

**Original Vision:**
> "A simple pluggable cache layer that normalizes LLM queries to reduce costs."

**Expected Complexity:**
```python
# Simple integration
from l0_cache import L0L3Gateway

gateway = L0L3Gateway(llm_client)
response = gateway.chat("cancel order 12345")  # That's it!
```

**Expected Infrastructure:**
- Single Redis instance
- Optional normalizer plugin
- Drop-in replacement for LLM client

**Expected Cost:** ~$200/mo for small deployments

---

### Where We Are Now (Week 4)

**Actual Reality:**
> "An enterprise-grade caching infrastructure with human-in-the-loop approval, multi-factor authentication, distributed Redis clusters, circuit breakers, audit logging, and 24/7 operational support."

**Actual Complexity:**

```
┌─────────────────────────────────────────────────────────────┐
│  L0 HEURISTIC CACHE - ACTUAL SYSTEM ARCHITECTURE            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SECURITY LAYER                                      │  │
│  │  - MFA (Okta, Auth0)                                 │  │
│  │  - RBAC (role-based access control)                  │  │
│  │  - Audit logs (tamper-proof)                         │  │
│  │  - WAF (web application firewall)                    │  │
│  │  - DDoS protection                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  HITL APPROVAL SERVICE                               │  │
│  │  - Web UI (React + Node.js)                          │  │
│  │  - Approval queue management                         │  │
│  │  - Safety checks (toxicity, phishing)                │  │
│  │  - Dual-approval workflow (high-risk)                │  │
│  │  - Versioned cache entries                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  REDIS SENTINEL CLUSTER                              │  │
│  │  - Primary (writes)                                  │  │
│  │  - Replica 1 (reads + failover)                      │  │
│  │  - Replica 2 (reads + failover)                      │  │
│  │  - Auto-failover (5 second detection)                │  │
│  │  - TLS encryption in transit                         │  │
│  │  - Encryption at rest                                │  │
│  │  - ACLs (read-only gateways, write-only HITL)        │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  GATEWAY LAYER (Edge Deployment)                     │  │
│  │  - Circuit breakers (prevent cascading failures)     │  │
│  │  - Request coalescing (prevent thundering herd)      │  │
│  │  - Health checks (Redis, LLM)                        │  │
│  │  - PII-safe logging                                  │  │
│  │  - Multi-provider LLM fallback                       │  │
│  │  - Rate limiting                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  MONITORING & ALERTING                               │  │
│  │  - Datadog / Grafana                                 │  │
│  │  - PagerDuty (on-call rotation)                      │  │
│  │  - CloudTrail (audit logs)                           │  │
│  │  - Security scanning (SAST, DAST)                    │  │
│  │  - Chaos engineering (monthly drills)                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Actual Infrastructure Requirements:**
- Redis Sentinel (1 primary + 2 replicas)
- HITL web application (UI + API + database)
- Authentication service (Okta, Auth0)
- Audit logging (CloudTrail, Splunk)
- Monitoring (Datadog, Grafana, PagerDuty)
- WAF (Cloudflare, AWS WAF)
- Multi-region deployment (for <1ms latency)

**Actual Operational Requirements:**
- 2-3 HITL approvers (24/7 coverage)
- On-call engineering team (incidents, failover)
- Security team (monitoring, penetration testing)
- Compliance team (GDPR, SOC2 audits)

**Gap Between Vision and Reality:** 🚨 **MASSIVE**

---

## Part 2: TRUE Total Cost of Ownership (TCO)

### Original Cost Estimates (Naive)

From `ARCHITECTURE_SCALE.md`:

| Deployment Pattern | Infrastructure Cost | Capacity |
|-------------------|---------------------|----------|
| Single-region | $200/mo | 50K req/sec |
| Multi-region | $2,000/mo | 3M req/sec |
| K8s edge | $20,000/mo | 30M req/sec |

**Problem:** These estimates only included Redis + compute. They ignored:
- Security infrastructure
- Operational labor
- Compliance costs
- Development costs

---

### ACTUAL Total Cost of Ownership (Realistic)

#### Scenario 1: Self-Hosted (Enterprise Customer)

**Infrastructure Costs (Monthly):**

| Component | Cost | Notes |
|-----------|------|-------|
| **Redis Sentinel** | $500 | Primary + 2 replicas (r6g.large) |
| **HITL Web App** | $200 | EC2 t3.medium + RDS |
| **Gateway Instances** | $1,000 | 5x t3.large across regions |
| **Load Balancer** | $100 | ALB |
| **Authentication (Okta)** | $300 | $3/user/mo x 100 employees |
| **Monitoring (Datadog)** | $500 | 10 hosts + APM |
| **Logging (Splunk)** | $300 | 10 GB/day |
| **WAF (Cloudflare)** | $200 | Enterprise plan |
| **Backup Storage** | $50 | S3 snapshots |
| **Bandwidth** | $150 | Data transfer |
| **TOTAL INFRASTRUCTURE** | **$3,300/mo** | **$39,600/year** |

**Operational Costs (Annual):**

| Role | Cost | Notes |
|------|------|-------|
| **HITL Approvers** | $150K | 2 FTE @ $75K (24/7 coverage) |
| **DevOps Engineer** | $150K | 1 FTE (maintenance, on-call) |
| **Security Engineer** | $80K | 0.5 FTE (audits, pen testing) |
| **Compliance Officer** | $40K | 0.25 FTE (GDPR, SOC2) |
| **TOTAL LABOR** | **$420K/year** | |

**One-Time Costs:**

| Item | Cost | Notes |
|------|------|-------|
| **Development (Sprint 4)** | $100K | 4 weeks @ $25K/week (2 engineers) |
| **Security Audit** | $30K | Penetration testing |
| **SOC2 Certification** | $50K | Initial audit + tooling |
| **Training** | $10K | HITL approver training |
| **TOTAL ONE-TIME** | **$190K** | |

**TOTAL YEAR 1 COST:** $39.6K (infra) + $420K (labor) + $190K (one-time) = **$649,600**

**TOTAL ONGOING ANNUAL COST:** $39.6K (infra) + $420K (labor) = **$459,600/year**

---

#### Scenario 2: SaaS (Anthropic/OpenAI Hosts It)

**Infrastructure Costs (Monthly):**

| Component | Cost | Notes |
|-----------|------|-------|
| **Redis Sentinel (Multi-region)** | $2,000 | 3 regions x (1 primary + 2 replicas) |
| **HITL Web App** | $500 | Auto-scaling, multi-region |
| **Gateway Instances** | $5,000 | 20x instances across 10 regions |
| **Load Balancer** | $500 | Global load balancing |
| **Authentication (Auth0)** | $1,000 | $10/user/mo x 100 HITL users |
| **Monitoring (Datadog)** | $2,000 | 50 hosts + APM + logs |
| **WAF + DDoS (Cloudflare)** | $500 | Enterprise plan |
| **Backup Storage** | $200 | S3 multi-region |
| **Bandwidth** | $1,000 | High traffic |
| **TOTAL INFRASTRUCTURE** | **$12,700/mo** | **$152,400/year** |

**Operational Costs (Annual):**

| Role | Cost | Notes |
|------|------|-------|
| **HITL Approvers** | $300K | 4 FTE (24/7, global coverage) |
| **DevOps Engineers** | $450K | 3 FTE (on-call rotation) |
| **Security Engineers** | $300K | 2 FTE (monitoring, incidents) |
| **Product Manager** | $150K | 1 FTE (roadmap, customers) |
| **Support Engineers** | $200K | 2 FTE (customer issues) |
| **TOTAL LABOR** | **$1.4M/year** | |

**One-Time Costs:**

| Item | Cost | Notes |
|------|------|-------|
| **Development (Sprint 4)** | $200K | 8 weeks @ $25K/week (4 engineers) |
| **Security Audit** | $50K | Comprehensive pen testing |
| **SOC2 + ISO 27001** | $150K | Multi-certification |
| **TOTAL ONE-TIME** | **$400K** | |

**TOTAL YEAR 1 COST:** $152K (infra) + $1.4M (labor) + $400K (one-time) = **$1.952M**

**TOTAL ONGOING ANNUAL COST:** $152K (infra) + $1.4M (labor) = **$1.552M/year**

---

### Cost Comparison Summary

| Model | Year 1 Cost | Ongoing Annual Cost | Customers Needed to Break Even* |
|-------|-------------|---------------------|--------------------------------|
| **Self-Hosted Enterprise** | $650K | $460K | 5-10 (@ $50-100K/year) |
| **SaaS (Hosted)** | $1.95M | $1.55M | 300-1500 (@ $1K-5K/year) |

*Assumes 30% gross margin

---

## Part 3: ROI Analysis - Does This Save Money?

### Customer Savings Calculation

**Scenario:** E-commerce company with 1M customer service queries/month

**Without L0 Cache:**
```
1M queries/mo × $0.02/query (GPT-4) = $20,000/mo = $240,000/year
```

**With L0 Cache (70% hit rate):**
```
700K cached queries × $0.0001/query (Redis) = $70/mo
300K L3 queries × $0.02/query = $6,000/mo
TOTAL: $6,070/mo = $72,840/year

SAVINGS: $240K - $72.8K = $167,160/year (70% reduction)
```

**BUT:** Must subtract TCO

**Self-Hosted TCO:** $460K/year
**Net Cost:** $167K savings - $460K TCO = **-$293K LOSS** ❌

**PROBLEM:** The operational cost exceeds the LLM savings for most customers!

---

### Break-Even Analysis

**How much LLM spend is needed to justify L0 cache?**

**Self-Hosted Model:**
- TCO: $460K/year
- Need to save at least $460K in LLM costs
- At 70% reduction: $460K / 0.70 = $657K current LLM spend
- Implies **33M queries/year** (2.75M/month)

**Conclusion:** Only viable for customers spending **>$650K/year on LLMs**

**SaaS Model (Customer Perspective):**
- SaaS cost: $5K/year (estimated)
- Need to save at least $5K in LLM costs
- At 70% reduction: $5K / 0.70 = $7.1K current LLM spend
- Implies **356K queries/year** (30K/month)

**Conclusion:** Viable for customers spending **>$7K/year on LLMs**

**MARKET SIZE:**
- Self-hosted: ~500 companies (Fortune 1000 + unicorn startups)
- SaaS: ~50,000 companies (mid-market + growth-stage startups)

---

## Part 4: Distribution Model Analysis

### Option 1: Open-Source Self-Hosted ❌ NOT VIABLE

**Packaging:**
```bash
pip install l0-cache
docker-compose up  # Redis + HITL UI
```

**Pros:**
- Low barrier to entry
- Community adoption
- Viral growth potential

**Cons:**
- **TOO COMPLEX** for small teams to operate
  - Need to configure Redis Sentinel
  - Need to set up HITL approvers (who??)
  - Need to configure MFA, RBAC, audit logs
  - Need 24/7 on-call for Redis failures
  - Need security expertise (ACLs, encryption, WAF)
- **No revenue model** (can't charge for open-source)
- **Support burden** (GitHub issues, Slack community)
- **Security liability** (users will misconfigure and blame us)

**Verdict:** ❌ **DO NOT PURSUE**
*"If this is brittle then it's useless" - and it WILL be brittle in untrained hands.*

---

### Option 2: Self-Hosted Enterprise (Commercial License) ⚠️ VIABLE

**Packaging:**
- Helm chart (Kubernetes deployment)
- Terraform modules (AWS/GCP/Azure)
- Docker Compose (single-node development)
- Enterprise support contract required

**Target Customers:**
- Fortune 500 companies
- Large banks, healthcare orgs (compliance requirements)
- Companies spending >$1M/year on LLMs

**Pricing Model:**
```
$50,000/year base + $5/1000 queries
or
$100,000/year unlimited (enterprise tier)
```

**Pros:**
- High ACV (Annual Contract Value)
- Customers have ops teams to manage it
- Enterprise customers EXPECT complexity
- Can charge for support, training, compliance

**Cons:**
- Small addressable market (~500 companies)
- Long sales cycles (6-12 months)
- High support burden (custom deployments)
- Requires enterprise sales team

**Break-Even:**
- Need 5-10 customers @ $50-100K/year
- Achievable in Year 2

**Verdict:** ⚠️ **VIABLE** but requires enterprise sales motion

---

### Option 3: SaaS (Fully Managed) ✅ HIGHLY VIABLE

**Packaging:**
- API endpoint (drop-in replacement for OpenAI SDK)
- Web dashboard (HITL approval UI)
- Usage-based pricing

**Example Integration:**
```python
# Before (direct OpenAI)
from openai import OpenAI
client = OpenAI(api_key="sk-...")
response = client.chat.completions.create(...)

# After (L0 Cache as a Service)
from l0_cache import L0CachedClient
client = L0CachedClient(
    llm_provider="openai",
    llm_api_key="sk-...",
    l0_api_key="l0-..."  # L0 Cache API key
)
response = client.chat.completions.create(...)  # Same interface!
```

**Target Customers:**
- Mid-market SaaS companies (100-1000 employees)
- AI-first startups
- Customer service platforms (Zendesk, Intercom competitors)
- Anyone using LLMs in production

**Pricing Model:**
```
Starter: $99/mo (100K queries)
Growth: $999/mo (1M queries)
Enterprise: $5,000/mo (10M queries + custom HITL)
```

**Pros:**
- **No operational burden for customers** (we handle everything)
- Large addressable market (~50K companies)
- Predictable revenue (SaaS ARR)
- Fast onboarding (API key + 10 lines of code)
- We control HITL (consistent quality)
- Network effects (shared cache across customers for generic intents)

**Cons:**
- High upfront investment ($2M Year 1)
- Need to scale infrastructure
- Responsible for security, compliance, uptime
- Customer trust (handling their LLM traffic)

**Revenue Potential:**
- Year 1: 100 customers @ $1K/mo = $1.2M ARR
- Year 2: 500 customers @ $2K/mo = $12M ARR
- Year 3: 2000 customers @ $3K/mo = $72M ARR (with growth tier upgrades)

**Break-Even:**
- Need ~130 customers @ $1K/mo to cover $1.55M annual cost
- Achievable in Year 2

**Verdict:** ✅ **HIGHLY VIABLE** with VC funding

---

### Option 4: Hybrid (SaaS + Self-Hosted Enterprise) ✅ BEST MODEL

**Strategy:**
1. **Start with SaaS** (get to market fast, validate demand)
2. **Add self-hosted** option in Year 2 (for enterprise customers who can't use SaaS due to compliance)

**Pricing Tiers:**

| Tier | Price | Model | Target Customer |
|------|-------|-------|-----------------|
| **Starter** | $99/mo | SaaS | Startups (<10K queries/mo) |
| **Growth** | $999/mo | SaaS | Growth-stage (100K-1M queries/mo) |
| **Enterprise SaaS** | $5K/mo | SaaS | Large companies (>1M queries/mo) |
| **Enterprise Self-Hosted** | $100K/year | On-prem | Banks, healthcare (compliance) |

**Pros:**
- Capture both markets
- SaaS revenue funds self-hosted development
- Self-hosted customers have highest ACV

**Cons:**
- More complex product offering
- Need to maintain two deployment models

**Verdict:** ✅ **BEST LONG-TERM MODEL**

---

## Part 5: Feasibility Assessment

### ❌ NOT Feasible As:
1. **Open-source self-hosted** - Too complex for small teams, no revenue model
2. **Simple drop-in library** - Requires enterprise infrastructure to run properly
3. **Side project** - Needs $2M+ investment and full team

### ⚠️ Marginally Feasible As:
1. **Self-hosted enterprise only** - Viable but small market, long sales cycles
2. **Consulting project** - Could deploy for 1-2 large customers, but not scalable

### ✅ HIGHLY Feasible As:
1. **SaaS (Fully Managed)** - Best product-market fit
2. **Hybrid (SaaS + Self-Hosted)** - Best long-term revenue model
3. **Anthropic/OpenAI Feature** - If Anthropic/OpenAI built this as a native feature

---

## Part 6: Key Decision Points

### Decision 1: Who Should Build This?

**Option A: Independent Startup**
- Pros: Own the market, capture full revenue
- Cons: Need $2M+ seed funding, compete with Anthropic/OpenAI
- Verdict: ⚠️ Possible but risky (Anthropic could crush you by building it natively)

**Option B: Anthropic/OpenAI Native Feature**
- Pros: Distribution, trust, native integration
- Cons: You don't own it, limited upside
- Verdict: ✅ **BEST STRATEGIC FIT** (pitch to Anthropic as a product feature)

**Option C: Enterprise Infrastructure Vendor (e.g., Redis Labs, Kong)**
- Pros: Existing sales motion, enterprise relationships
- Cons: Not core to their business
- Verdict: ⚠️ Possible partnership/acquisition target

**RECOMMENDATION:** Pitch to Anthropic as a **native caching feature** before building a standalone company.

---

### Decision 2: If Building as Startup, What's the MVP?

**Phase 1: SaaS MVP (3 months, $200K)**
- Basic L0 cache (Redis + Gateway)
- HITL approval UI (simple web app)
- API (drop-in replacement for OpenAI SDK)
- Single-region deployment
- Manual security (no MFA, basic RBAC)
- Target: 10 beta customers

**Phase 2: Production-Ready (6 months, $500K)**
- Multi-region deployment
- Full security (MFA, RBAC, audit logs)
- Circuit breakers, health checks
- SOC2 compliance
- Target: 50 paying customers

**Phase 3: Scale (12 months, $1M)**
- Auto-scaling
- Multi-provider LLM support
- Advanced analytics
- Self-hosted option (enterprise)
- Target: 200 paying customers

**TOTAL TO SERIES A:** $1.7M over 21 months

---

### Decision 3: What's the Market Timing?

**Indicators This is the RIGHT Time:**
- ✅ LLM costs are top concern for companies (GPT-4 is expensive)
- ✅ Production LLM usage is exploding (customer service, coding assistants)
- ✅ Companies are building in-house caching (reinventing the wheel)
- ✅ No dominant player in LLM caching yet

**Indicators This is the WRONG Time:**
- ⚠️ OpenAI might build this natively (prompt caching exists but is limited)
- ⚠️ LLM costs are dropping (GPT-5 might be cheaper)
- ⚠️ Companies already built in-house solutions (switching cost)

**Verdict:** ✅ **WINDOW IS OPEN** but closing (12-18 month opportunity before LLM providers build it natively)

---

## Part 7: Final Recommendations

### Recommendation 1: Strategic Direction

**DO NOT BUILD AS OPEN-SOURCE PROJECT**
- Too complex for self-hosting
- No revenue model
- High support burden

**DO BUILD AS SaaS IF:**
- You can raise $2M seed round
- You can get to market in <6 months
- You can acquire 100 customers in Year 1

**OR BETTER YET: PITCH TO ANTHROPIC/OPENAI**
- Position as a native feature (like prompt caching, but better)
- Revenue model: 10% markup on cached queries
- Example: Cache hit costs $0.0001 instead of free, cache miss costs normal price
- Anthropic makes money, customers save 99.5% on cache hits

---

### Recommendation 2: If Building as Startup

**Go-to-Market Strategy:**

**Month 1-3: MVP**
- Build basic SaaS (no security sprint, just core functionality)
- Target 5 beta customers (high-trust relationships)
- Manually operate HITL (founder approves all entries)
- Deploy single-region only

**Month 4-6: Beta**
- Add basic security (API keys, simple RBAC)
- Target 20 beta customers
- Hire 1 HITL approver
- Add health checks, basic monitoring

**Month 7-9: Launch**
- Full security sprint (MFA, audit logs, Redis Sentinel)
- SOC2 Type 1
- Target 50 paying customers
- Hire devops engineer

**Month 10-12: Scale**
- Multi-region deployment
- Target 100 customers
- Series A fundraise ($10M)

**Key Metrics:**
- Month 6: $10K MRR (20 customers @ $500/mo)
- Month 9: $50K MRR (50 customers @ $1K/mo)
- Month 12: $120K MRR (100 customers @ $1.2K/mo) → $1.44M ARR

**Fundraising Milestones:**
- Pre-seed ($500K): To build MVP (Month 0)
- Seed ($2M): After beta (Month 6, $10K MRR)
- Series A ($10M): After launch (Month 12, $1.44M ARR)

---

### Recommendation 3: If Pitching to Anthropic/OpenAI

**Value Proposition:**
> "We reduce customer LLM costs by 70% while increasing Anthropic revenue."

**How:**
- Customers enable L0 caching (opt-in)
- Cache hits cost $0.0001 (instead of $0.02 for GPT-4)
- Customers save $0.0199 per cache hit (99.5% savings)
- Anthropic earns $0.0001 per cache hit (instead of $0 with current prompt caching)
- **Anthropic makes money on cache hits, customers still save 99.5%**

**Revenue Model:**
- Without L0: 1M queries × $0.02 = $20,000 revenue
- With L0 (70% hit rate):
  - 700K cache hits × $0.0001 = $70
  - 300K cache misses × $0.02 = $6,000
  - **Total: $6,070 revenue (30.5% of original)**

**BUT:** Customer retention and usage goes UP because costs are lower
- Customers use 3x more LLM (can afford it now)
- Net revenue: 3M queries with L0 vs 1M without = **$18,210 vs $20,000**
- Only 9% revenue decrease but **3x usage increase** (stickiness, market share)

**Pitch:**
1. **Strategic moat:** Competitors (OpenAI) don't have semantic caching
2. **Customer retention:** Customers use more Claude (lower costs)
3. **New revenue stream:** Charge for cache hits (currently free with prompt caching)
4. **Compliance:** Built-in HITL ensures responses meet enterprise standards

---

## Conclusion

### The Answer: **IT DEPENDS ON WHO BUILDS IT**

| Builder | Viability | Recommended Path |
|---------|-----------|------------------|
| **Anthropic/OpenAI** | ✅ HIGHLY VIABLE | Build as native feature, charge for cache hits |
| **Funded Startup** | ✅ VIABLE | SaaS model, $2M seed, 12-month timeline to 100 customers |
| **Bootstrapped Startup** | ⚠️ MARGINALLY VIABLE | Self-hosted enterprise only, slow growth |
| **Open-Source Project** | ❌ NOT VIABLE | Too complex to self-host, no revenue model |
| **Side Project** | ❌ NOT VIABLE | Requires $2M+ investment |

---

### Final Verdict

**YES, THE PRODUCT IS STILL FEASIBLE** but with critical caveats:

1. **Not as a "simple plugin"** - It's enterprise infrastructure
2. **Not for small teams** - Requires operational expertise
3. **Not open-source** - Too complex for self-hosting
4. **Best as SaaS** - Fully managed, usage-based pricing
5. **Or best as Anthropic feature** - Native integration, built-in distribution

**The complexity we uncovered is REAL and UNAVOIDABLE.** The question is: who operates it?

**RECOMMENDED NEXT STEPS:**

1. **Reach out to Anthropic** (partnerships/product team) with this analysis
2. **If Anthropic not interested:** Raise $2M seed to build SaaS
3. **If can't raise:** Don't build it (the operational complexity requires capital)

**The market opportunity is real ($5-20M ARR potential), but it's not a side project.**
