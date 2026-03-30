# VPS Provider Comparison — SitoPresepi2

**Author:** Patrick (Solution Architect)  
**Date:** 2026-07-17  
**Purpose:** Evaluate European VPS providers against SitoPresepi2's requirements  
**Status:** Draft — pending Salvatore (DevOps) enrichment

---

## Context

SitoPresepi2 is currently locked to **DigitalOcean Frankfurt, 2-core/4GB** per the constitution. This document evaluates whether that remains the optimal choice by comparing five EU-friendly VPS providers against the project's actual requirements.

### Requirements Profile

| Requirement | Detail |
|---|---|
| Workload | Django + Next.js + PostgreSQL 16 + Nginx, all containerised (Docker Compose) |
| Baseline spec | 2 vCPU, 4 GB RAM, ~40–80 GB SSD |
| Location | EU (Frankfurt preferred, any EU DC acceptable for GDPR) |
| Traffic | Low — small artisan e-commerce, ~1000s of visits/month expected |
| Managed DB | Nice-to-have, not required (PostgreSQL runs in container) |
| IaC | Terraform/API desirable for reproducibility |
| Budget target | < €25/mo for the primary instance |

---

## Summary Comparison

| Provider | HQ | EU DCs | Closest Plan | vCPU | RAM | SSD | Price/mo | Traffic | GDPR Entity |
|---|---|---|---|---|---|---|---|---|---|
| **Hetzner** | DE | Nuremberg, Falkenstein, Helsinki | CPX21 | 3 AMD | 4 GB | 80 GB | **€8.99** | 20 TB | German GmbH |
| **Scaleway** | FR | Paris, Amsterdam | DEV1-M | 3 | 4 GB | — | **€14.45** | Included (no egress) | French SAS |
| **OVHcloud** | FR | Multiple EU | VPS-1 | 4 | 8 GB | 75 GB | **~€6** | Unlimited (3 Gbps) | French SAS |
| **UpCloud** | FI | Frankfurt, Amsterdam, Helsinki + 12 more | Developer 4GB | 2 | 4 GB | 60 GB | **€18** | Zero-cost egress | Finnish Oy |
| **DigitalOcean** | US | Frankfurt, Amsterdam, London | Basic 4GiB | 2 | 4 GB | 80 GB | **$24 (~€22)** | — | US Inc. |

*Prices as fetched 2026-07-17. VAT treatment varies.*

---

## Per-Provider Analysis

### 1. Hetzner Cloud

**Best value. German HQ. Strong EU GDPR position.**

| Plan | vCPU | RAM | SSD | €/mo |
|---|---|---|---|---|
| CPX11 | 2 AMD | 2 GB | 40 GB | 4.49 |
| **CPX21** | **3 AMD** | **4 GB** | **80 GB** | **8.99** |
| CPX31 | 4 AMD | 8 GB | 160 GB | 15.99 |
| CPX41 | 8 AMD | 16 GB | 240 GB | 29.99 |

- **Shared vCPU** (Regular Performance line) — adequate for our workload
- 20 TB traffic included — more than sufficient
- DCs: Nuremberg, Falkenstein (both Germany), Helsinki
- ISO 27001 certified
- REST API, CLI, Terraform provider, Ansible integration
- Docker CE available as one-click app
- Backups: from €0.60/mo (automated daily, 7 slots)
- Snapshots: €0.011/GB/mo
- Block Storage, Object Storage, Load Balancers available as add-ons
- **No managed PostgreSQL** — DB must run in container or self-managed

**Architectural assessment:** At €8.99/mo for 3 vCPU / 4 GB / 80 GB, Hetzner delivers 2.5× the value of DigitalOcean for a comparable or better spec. German entity simplifies GDPR compliance. The extra vCPU and storage headroom at this price point is significant.

### 2. Scaleway

**EU-native. Competitive pricing. Managed PostgreSQL available.**

| Plan (Dev) | vCPU | RAM | €/mo |
|---|---|---|---|
| DEV1-S | 2 | 2 GB | 6.42 |
| **DEV1-M** | **3** | **4 GB** | **14.45** |
| DEV1-L | 4 | 8 GB | 30.66 |

| Plan (GP Shared) | vCPU | RAM | €/mo |
|---|---|---|---|
| BASIC2-A2C-4G | 2 | 4 GB | 16.79 |
| BASIC2-A4C-8G | 4 | 8 GB | 37.74 |

- **No egress fees** — all traffic included
- DCs: Paris (PAR-1), Amsterdam
- French SAS, full EU data sovereignty
- Managed PostgreSQL available (separate pricing)
- API, CLI, Terraform provider
- **Storage**: local storage not shown in Dev plans — likely uses Block Storage (separate cost)

**Architectural assessment:** DEV1-M at €14.45 is reasonable but ~60% more expensive than Hetzner for equivalent specs. The "no egress" policy is attractive but irrelevant at our traffic levels. Storage costs need investigation — Dev instances may require paid Block Storage on top.

### 3. OVHcloud

**Aggressive pricing. Unlimited traffic. EU entity.**

- VPS-1: 4 vCores, 8 GB RAM, 75 GB SSD — **~€6/mo**
- VPS-2: 6 vCores, 12 GB RAM — ~€10/mo
- Unlimited traffic up to 3 Gbps
- Daily backup included
- Anti-DDoS included
- French SAS (Roubaix HQ)
- API, Terraform, CLI available
- Managed PostgreSQL available

**Architectural assessment:** On paper, OVHcloud offers the most resources per euro — 4 vCores and 8 GB RAM for ~€6 is extraordinary. However, OVH has a reputation for inconsistent support quality and occasional infrastructure incidents. The price is hard to argue with for a low-traffic artisan site. Worth serious consideration if support SLA isn't critical.

### 4. UpCloud

**Finnish. Premium positioning. 99.99% SLA.**

| Plan | vCPU | RAM | SSD | €/mo |
|---|---|---|---|---|
| Developer 4GB | 2 | 4 GB | 60 GB | 18.00 |
| General Purpose (varies) | — | — | — | Higher |

- Zero-cost egress
- 15 global DCs including Frankfurt, Amsterdam, Helsinki
- Finnish Oy — strong EU GDPR position
- 99.99% SLA (highest among those compared)
- Managed PostgreSQL: 1GB/1CPU/10GB = €8/mo, 4GB/2CPU/50GB = €60/mo
- API available

**Architectural assessment:** At €18/mo, UpCloud is 2× Hetzner for similar specs. The 99.99% SLA is impressive but likely not worth the premium for a low-traffic artisan site. Managed PostgreSQL pricing is steep. UpCloud is better suited for production workloads with stricter uptime requirements.

### 5. DigitalOcean (Current)

**US HQ. Largest ecosystem. Currently in constitution.**

| Plan | vCPU | RAM | SSD | $/mo |
|---|---|---|---|---|
| Basic 4GiB | 2 | 4 GB | 80 GB | $24 |
| Basic 8GiB | 4 | 8 GB | 160 GB | $48 |
| General Purpose 8GiB | 2 | 8 GB | 25 GB | $63 |

- EU DCs: Frankfurt, Amsterdam, London
- Per-second billing (since Jan 2026)
- Managed databases available
- Extensive documentation and community
- US-incorporated — data processing under EU-US DPF

**Architectural assessment:** DigitalOcean remains a solid, well-documented platform with excellent developer experience. However, at $24/mo (~€22) it is the most expensive option for 2 vCPU / 4 GB. Being US-incorporated adds a layer of GDPR complexity (requires EU-US Data Privacy Framework reliance). The ecosystem and documentation advantage is real but less relevant for a small, well-defined deployment.

---

## Architect's Ranking

| Rank | Provider | €/mo | Rationale |
|---|---|---|---|
| **1** | **Hetzner** | **8.99** | Best price-performance. German entity (GDPR simplicity). Excellent API/IaC. Only downside: no managed PostgreSQL (irrelevant — we run in Docker). |
| **2** | OVHcloud | ~6.00 | Cheapest. More resources. EU entity. Support quality is the concern. |
| **3** | Scaleway | 14.45 | Solid EU option. No egress fees. Storage costs unclear for Dev tier. |
| **4** | UpCloud | 18.00 | Premium SLA. Good product. Price premium not justified at our scale. |
| **5** | DigitalOcean | ~22.00 | Current choice. Good DX. Most expensive. US entity adds GDPR friction. |

### Preliminary Recommendation

**Hetzner CPX21 (€8.99/mo)** as the primary candidate, replacing DigitalOcean.

**Key arguments:**
1. **60% cost reduction** (€22 → €9/mo) with equal or better specs (3 vCPU vs 2)
2. **German GmbH** — simplest possible GDPR position for an EU site handling EU personal data
3. **ISO 27001** — meets security baseline without additional effort
4. **Full IaC support** — Terraform, API, CLI — matches our reproducibility goals
5. **20 TB traffic** — vastly more than needed, eliminates any surprise bandwidth costs

**Risk factors to evaluate (Salvatore's section below):**
- Docker Compose workflow compatibility
- Backup/DR strategy on Hetzner vs DigitalOcean
- Monitoring and alerting options
- Migration effort from DigitalOcean (if already provisioned)
- Network performance (latency to Italian/UK/German users)

---

## DevOps Perspective

**Author:** Salvatore (DevOps Engineer)  
**Date:** 2026-07-17

### Per-Provider Operational Assessment

#### Hetzner Cloud

- **Docker Compose:** Full support. Ubuntu/Debian images ship with apt-installable Docker CE. One-click Docker app available in marketplace. No proprietary container runtime — standard Docker Engine.
- **Backup/DR:** Automated daily backups (7 slots, oldest rotated) from €0.60/mo. Snapshots at €0.011/GB/mo for point-in-time captures. No cross-DC replication built in — must script snapshot transfer or use external backup (e.g., restic to Hetzner Object Storage at €0.006/GB/mo).
- **Monitoring:** No built-in monitoring dashboard. Hetzner provides basic metrics (CPU, disk, network) in the console. For production: deploy Prometheus + Grafana in a sidecar container or use external (e.g., Uptime Robot, Betterstack). This is standard for any VPS provider at this tier.
- **CI/CD:** REST API + CLI (`hcloud`) + Terraform provider (`hetznercloud/hcloud`). GitHub Actions integration straightforward — SSH deploy or API-driven. No proprietary CI/CD lock-in.
- **Network/CDN:** No built-in CDN. Floating IPs (€1/mo IPv6), private networks (free), firewalls (free, stateful). For CDN: Cloudflare free tier in front, which we'd use regardless of provider.
- **Operational notes:** Hetzner's API is clean and well-documented. `hcloud` CLI is excellent for scripting. Community tutorials are extensive. Support is email-only on cloud tier (no live chat) — adequate for our needs.

#### Scaleway

- **Docker Compose:** Full support on all instance types. Standard Ubuntu/Debian images. No proprietary restrictions.
- **Backup/DR:** Automated backups available. Snapshots supported. Object Storage (S3-compatible) for offsite backup. Managed PostgreSQL includes automated backups.
- **Monitoring:** Cockpit (built-in observability platform) — includes metrics, logs, and alerting. This is a differentiator vs Hetzner at no extra cost.
- **CI/CD:** API, CLI (`scw`), Terraform provider. GitHub Actions compatible. Container Registry available if we ever need to push images.
- **Network/CDN:** No built-in CDN. Private networks, security groups available. Egress is free — simplifies cost modelling.
- **Operational notes:** Dev instances use ephemeral local storage — **data loss risk on instance stop/start**. This is a critical concern: the DEV1-M plan likely requires attaching Block Storage (€0.08/GB/mo = ~€3.20 for 40 GB) for persistence, pushing effective cost to ~€17.65/mo. General Purpose instances (BASIC2) include persistent storage but start at €16.79. The Cockpit monitoring is genuinely useful.

#### OVHcloud

- **Docker Compose:** Full support. Standard Linux images. No restrictions.
- **Backup/DR:** Daily backup included in VPS price (significant advantage). Snapshots available. Object Storage (S3-compatible) for offsite.
- **Monitoring:** Basic metrics in control panel. No built-in observability stack. Same Prometheus/Grafana self-deploy as Hetzner.
- **CI/CD:** API, Terraform provider, CLI (ovh-cli). Slightly less mature tooling ecosystem than Hetzner or DigitalOcean.
- **Network/CDN:** Anti-DDoS included (significant at this price). No built-in CDN. Private networks available.
- **Operational notes:** OVH has a **mixed operational reputation**. The 2021 Strasbourg DC fire (SBG2) demonstrated real DR risks. They've rebuilt, but it's a factor. Support responsiveness varies. The VPS-1 at ~€6/mo with 4 vCores and 8 GB seems too good — historically OVH VPS plans have variable CPU performance under contention. Daily backup included is excellent. Anti-DDoS included is a plus. API tooling is functional but less polished than Hetzner's.

#### UpCloud

- **Docker Compose:** Full support. MaxIOPS storage is genuinely fast — better disk I/O than most competitors.
- **Backup/DR:** Automated backups available (paid). Storage backup and server cloning. No S3-compatible object storage — use external.
- **Monitoring:** Basic server metrics in console. No built-in observability stack. Self-deploy Prometheus/Grafana.
- **CI/CD:** API available. Terraform provider exists but smaller community. CLI less mature.
- **Network/CDN:** No built-in CDN. Private networking available. Zero-cost egress.
- **Operational notes:** UpCloud's MaxIOPS storage is their genuine differentiator — measurably faster disk I/O than Hetzner or DO. The 99.99% SLA is backed contractually. However, at €18/mo for 2 vCPU / 4 GB, the premium doesn't align with our workload (low I/O artisan e-commerce). Smaller community means fewer tutorials and community answers.

#### DigitalOcean

- **Docker Compose:** Full support. One-click Docker droplet available. Extensive documentation.
- **Backup/DR:** Automated weekly backups (20% of droplet price = ~$4.80/mo). Snapshots at $0.06/GB/mo. Spaces (S3-compatible) for offsite.
- **Monitoring:** Built-in monitoring with alerting (CPU, memory, disk, bandwidth). Free. Better than Hetzner's basic metrics but less comprehensive than Scaleway's Cockpit.
- **CI/CD:** Excellent API, CLI (`doctl`), Terraform provider. GitHub Actions integration well-documented. App Platform available (not needed but shows ecosystem depth).
- **Network/CDN:** Built-in CDN via Spaces CDN. Cloud firewalls. VPC. Load balancers.
- **Operational notes:** Best developer experience of all five. Documentation is industry-leading. Community tutorials cover every conceivable scenario. The $24/mo price for 2 vCPU / 4 GB is the clear downside. Backups at 20% of droplet price adds up. US entity means reliance on EU-US Data Privacy Framework for GDPR — additional legal surface area.

### Migration Assessment

**Current state:** No VPS has been provisioned yet (infrastructure.md confirms "planned — owner evaluating options"). This means there is **zero migration cost** — we're choosing a greenfield deployment target.

**Setup effort comparison:**

| Provider | Initial Setup Effort | Notes |
|---|---|---|
| Hetzner | Low | `hcloud` CLI or Terraform. Docker CE one-click. SSH key upload via API. |
| Scaleway | Low | `scw` CLI or Terraform. Standard setup. Watch for Block Storage attachment on Dev instances. |
| OVHcloud | Medium | API works but slightly more steps. Control panel UX is dated. |
| UpCloud | Low | API-driven. Clean console. |
| DigitalOcean | Lowest | Best onboarding UX. `doctl` is excellent. Most tutorials available. |

Since no infrastructure exists yet, the "migration" question is moot. All five can be provisioned in under an hour with Docker Compose up and running.

### DevOps Ranking

| Rank | Provider | €/mo (effective) | Rationale |
|---|---|---|---|
| **1** | **Hetzner** | **~€9.60** (instance + backup) | Best cost/spec ratio. Clean API. Terraform-ready. German DC and entity. No managed PG needed (Docker container). Backup adds ~€0.60. |
| **2** | OVHcloud | **~€6** (backup included) | Cheapest all-in. Daily backup included. Anti-DDoS included. Operational reputation is the risk factor. |
| **3** | DigitalOcean | **~€29** (instance + backup) | Best DX and docs. Most expensive. US entity GDPR friction. Backup pricing (20% of instance) is aggressive. |
| **4** | Scaleway | **~€17.65** (instance + block storage) | Good product but Dev tier storage gotcha pushes real cost up. Cockpit monitoring is a plus. |
| **5** | UpCloud | **€18+** | Good product, premium pricing not justified at our scale. Smaller tooling ecosystem. |

### Combined Recommendation

Patrick and I converge on the same conclusion: **Hetzner CPX21** is the optimal choice for SitoPresepi2.

**From an operations standpoint:**

1. **€9.60/mo all-in** (instance + automated backup) vs €29/mo on DigitalOcean — 67% cost reduction
2. **3 AMD vCPUs / 4 GB / 80 GB** — more headroom than DO's 2 vCPU at less than half the price
3. **German entity + German DCs** — cleanest GDPR posture for an Italian-owned site serving IT/EN/DE users
4. **Terraform + `hcloud` CLI** — fully automatable provisioning, compatible with our GitHub Actions deployment pipeline
5. **No managed PostgreSQL needed** — our PostgreSQL runs in Docker Compose; Hetzner's lack of managed DB is irrelevant
6. **20 TB traffic** — eliminates any bandwidth surprise; DO doesn't even list included traffic clearly at this tier

**Deployment plan sketch (if approved):**
1. Provision CPX21 in Nuremberg or Falkenstein via Terraform
2. Enable automated backups (€0.60/mo)
3. Configure Hetzner Cloud Firewall (free, stateful — replaces DO's cloud firewall)
4. SSH key deployment via `hcloud`
5. Docker Compose up (Django + Next.js + PostgreSQL + Nginx)
6. Cloudflare DNS + free CDN in front
7. Total estimated monthly cost: **~€10** (instance + backup)

**Fallback:** OVHcloud VPS-1 at ~€6/mo if budget is the primary constraint and the owner accepts the operational reputation trade-off.

**Status:** Owner has approved Hetzner CPX21 as the target. No server will be provisioned until a working website is ready for deployment. Constitution amendment and `infrastructure.md` update deferred to provisioning time.

**At provisioning, remember:**
- Nuremberg (NBG1) or Falkenstein (FSN1) DC
- Ubuntu 24.04 LTS image
- SSH key uploaded at creation, password auth disabled
- Primary IPv4 (+~€0.50/mo)
- Automated Backups (+~€1.80/mo)
- Cloud Firewall: allow 22, 80, 443 inbound only
- Install Docker Engine via official apt repo (not one-click app)
- **Estimated monthly cost: ~€11.30**
