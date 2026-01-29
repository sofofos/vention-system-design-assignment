# Budget Assumptions (Updated)

*Updated to reflect final technology choices from Video Upload System Design*

## Final Stack Summary

| Layer | Choice | Pricing Model |
|-------|--------|---------------|
| Object Storage | DigitalOcean Spaces | $5/250GB + $0.02/GB overage |
| Transcoding | Coconut.co | $0.015/output minute |
| Background Jobs | Sidekiq + DO Managed Valkey | $15+/month |
| Database | DO Managed PostgreSQL | $15+/month |
| Compute | DO Droplets | Starting at $4/month |
| CDN | DO CDN (included with Spaces) | Included |

---

## Usage Assumptions

| Metric | Value |
|--------|-------|
| DAU | 20,000 - 50,000 |
| Daily uploads (~5% of DAU) | 1,000 - 2,500 |
| Average video length | 5 minutes |
| Output resolutions | 4 (360p, 480p, 720p, 1080p) |
| Average file size (original) | ~200MB |
| Storage per video (original + HLS) | ~400MB |

---

## Back-of-Envelope Monthly Costs

### Storage (DigitalOcean Spaces)

**Calculation:**
- New storage/day: 1,000-2,500 uploads × 400MB = 400GB - 1TB/day
- Monthly accumulation: ~12-30TB new data
- After 6 months: ~75-180TB total stored

| Storage Level | Calculation | Monthly Cost |
|---------------|-------------|--------------|
| Month 1 (~20TB) | $5 + (20,000 - 250) × $0.02 | ~$400 |
| Month 6 (~100TB) | $5 + (100,000 - 250) × $0.02 | ~$2,000 |

**Estimate: $400 - $2,000/month** (grows over time)

> Source: [DigitalOcean Spaces Pricing](https://docs.digitalocean.com/products/spaces/details/pricing/)

---

### Transcoding (Coconut.co)

**Calculation:**
- Input: 1,000-2,500 uploads/day × 5 min = 5,000-12,500 min/day
- Output (4 resolutions): 20,000-50,000 output min/day
- Monthly: 600,000 - 1,500,000 output minutes
- Standard rate: $0.015/min

| Scenario | Output Minutes/Month | Cost (Standard) | Cost (Volume Discount*) |
|----------|---------------------|-----------------|------------------------|
| Lower (1k uploads/day) | 600,000 | $9,000 | ~$5,000-$6,000 |
| Upper (2.5k uploads/day) | 1,500,000 | $22,500 | ~$12,000-$15,000 |

*Volume pricing available for 50k+ min/month - contact Coconut for custom rates*

**Estimate: $5,000 - $15,000/month** (with expected volume discount)

> Source: [Coconut.co Pricing](https://www.coconut.co/pricing)

---

### Compute (DigitalOcean Droplets)

| Component | Spec | Count | Unit Cost | Total |
|-----------|------|-------|-----------|-------|
| API Servers | 2 vCPU, 4GB RAM | 2-3 | $24/mo | $48-72 |
| Sidekiq Workers | 2 vCPU, 4GB RAM | 2-4 | $24/mo | $48-96 |
| **Subtotal** | | | | **$96-168** |

**Estimate: $100 - $200/month**

> Source: [DigitalOcean Droplet Pricing](https://www.digitalocean.com/pricing/droplets)

---

### Database (DO Managed PostgreSQL)

| Configuration | Use Case | Cost |
|---------------|----------|------|
| Basic (1GB RAM, single node) | Dev/staging | $15/mo |
| Production (4GB RAM, HA) | Production | $80-120/mo |
| With read replica | Multi-region | $160-240/mo |

**Estimate: $80 - $240/month**

> Source: [DigitalOcean Managed Database Pricing](https://www.digitalocean.com/pricing/managed-databases)

---

### Redis/Valkey (for Sidekiq)

| Configuration | Cost |
|---------------|------|
| Basic (1GB RAM) | $15/mo |
| Production (2GB RAM) | $30-60/mo |

**Estimate: $30 - $60/month**

> Source: [DigitalOcean Managed Database Pricing](https://www.digitalocean.com/pricing/managed-databases)

---

### Data Transfer

- Spaces includes 1TB free outbound transfer
- CDN included with Spaces at no extra cost
- Overage: $0.01/GB

| Scenario | Estimated Egress | After Free Tier | Cost |
|----------|------------------|-----------------|------|
| Lower | ~5TB/month | 4TB | $40 |
| Upper | ~15TB/month | 14TB | $140 |

**Estimate: $50 - $150/month**

> Source: [DigitalOcean Bandwidth Billing](https://docs.digitalocean.com/platform/billing/bandwidth/)

---

### Monitoring & Misc

- Application monitoring (Datadog/NewRelic free tier or self-hosted)
- Error tracking (Sentry free tier)
- Logging

**Estimate: $100 - $300/month**

---

## Cost by Scale Tier

### Tier 1: Launch Phase
**20k DAU | ~1,000 uploads/day | Month 1-2**

| Component | Calculation | Cost |
|-----------|-------------|------|
| Storage | ~20TB accumulated | $400 |
| Transcoding | 600k output min × $0.015 | $9,000 |
| Transcoding (w/ volume discount) | ~35% off | **$5,850** |
| Compute | 2 API + 2 workers | $96 |
| Database | Single node production | $60 |
| Redis/Valkey | Basic | $15 |
| Data Transfer | ~3TB over free tier | $30 |
| Monitoring | Basic setup | $100 |
| **Total** | | **~$6,550/month** |

**Cost per upload: ~$0.22** | **Cost per DAU: ~$0.33**

---

### Tier 2: Growth Phase
**35k DAU | ~1,750 uploads/day | Month 3-4**

| Component | Calculation | Cost |
|-----------|-------------|------|
| Storage | ~50TB accumulated | $1,000 |
| Transcoding | 1.05M output min × $0.015 | $15,750 |
| Transcoding (w/ volume discount) | ~40% off | **$9,450** |
| Compute | 2 API + 3 workers | $120 |
| Database | HA setup | $100 |
| Redis/Valkey | Production | $30 |
| Data Transfer | ~8TB over free tier | $80 |
| Monitoring | Enhanced | $150 |
| **Total** | | **~$10,930/month** |

**Cost per upload: ~$0.21** | **Cost per DAU: ~$0.31**

---

### Tier 3: Scale Phase
**50k DAU | ~2,500 uploads/day | Month 5-6**

| Component | Calculation | Cost |
|-----------|-------------|------|
| Storage | ~100TB accumulated | $2,000 |
| Transcoding | 1.5M output min × $0.015 | $22,500 |
| Transcoding (w/ volume discount) | ~45% off | **$12,375** |
| Compute | 3 API + 4 workers | $168 |
| Database | HA + read replica | $200 |
| Redis/Valkey | Production HA | $60 |
| Data Transfer | ~14TB over free tier | $140 |
| Monitoring | Full observability | $250 |
| **Total** | | **~$15,193/month** |

**Cost per upload: ~$0.20** | **Cost per DAU: ~$0.30**

---

## Summary: Cost Trajectory Over 6 Months

```
Monthly Cost
    │
$15k┤                                    ┌────── Tier 3: ~$15,200
    │                              ┌─────┘       (50k DAU)
    │                        ┌─────┘
$11k┤                  ┌─────┘                   Tier 2: ~$10,900
    │            ┌─────┘                         (35k DAU)
    │      ┌─────┘
 $7k┤──────┘                                     Tier 1: ~$6,550
    │                                            (20k DAU)
    └────┬────┬────┬────┬────┬────┬────
        M1   M2   M3   M4   M5   M6
```

| Tier | DAU | Uploads/day | Monthly Cost | Cost/Upload | Cost/DAU |
|------|-----|-------------|--------------|-------------|----------|
| 1 - Launch | 20k | 1,000 | **~$6,550** | $0.22 | $0.33 |
| 2 - Growth | 35k | 1,750 | **~$10,930** | $0.21 | $0.31 |
| 3 - Scale | 50k | 2,500 | **~$15,193** | $0.20 | $0.30 |

**Key insight:** Costs scale ~linearly with usage, but unit economics improve slightly at scale due to volume discounts.

---

## Cost Breakdown by Category (at Scale)

```
Transcoding ████████████████████████████████████  81%  ($12,375)
Storage     █████████                             13%  ($2,000)
Infra       ███                                    6%  ($818)
            └────────────────────────────────────────────────
```

**Transcoding dominates at ~81% of total cost** - this is the lever to optimize.

---

## Key Differences from Original Estimate

| Component | Original Estimate | Updated Estimate | Notes |
|-----------|-------------------|------------------|-------|
| Storage | $1,500-$3,000 | $400-$2,000 | DO Spaces cheaper than S3/GCS |
| Transcoding | $2,000-$5,000 | $5,000-$15,000 | **Higher** - 4 output resolutions × $0.015/min |
| Compute | $500-$1,500 | $100-$200 | **Lower** - DO Droplets very cost-effective |
| Database | $200-$500 | $80-$240 | **Lower** - DO managed DB affordable |
| Transfer | $300-$800 | $50-$150 | **Lower** - DO CDN included, generous free tier |
| **Total** | **$5,000-$12,000** | **$5,760-$17,950** | Higher ceiling due to transcoding |

---

## Cost Optimization Opportunities

1. **Transcoding costs are dominant** - Contact Coconut.co for volume pricing (50k+ min/month qualifies)
2. **Selective transcoding** - Only generate resolutions ≤ source quality
3. **Cold storage** - Move old originals to DO Spaces Cold Storage ($0.007/GB vs $0.02/GB)
4. **On-demand transcoding** - Only transcode high resolutions when actually requested (deferred optimization)

---

## Revised Budget Assumption

> **Budget Assumption by Phase:**
> - **Launch (20k DAU):** ~$6,500/month
> - **Growth (35k DAU):** ~$11,000/month
> - **Scale (50k DAU):** ~$15,000/month
>
> Transcoding is the dominant cost (~81%). Negotiate Coconut.co volume pricing early—you'll qualify for custom rates from month 1. The DigitalOcean stack keeps infrastructure costs low (~$800-1,200/month), making this architecture affordable for a funded startup.

---

## References

- [DigitalOcean Spaces Pricing](https://docs.digitalocean.com/products/spaces/details/pricing/)
- [Coconut.co Pricing](https://www.coconut.co/pricing)
- [DigitalOcean Droplet Pricing](https://www.digitalocean.com/pricing/droplets)
- [DigitalOcean Managed Database Pricing](https://www.digitalocean.com/pricing/managed-databases)
- [DigitalOcean Bandwidth Billing](https://docs.digitalocean.com/platform/billing/bandwidth/)
