# Cost management

This lab is built so that every billable piece is small, tagged, and easy to delete.
Prices below are **reference rates checked in September 2026**. AWS prices change and vary by region, so always confirm on the official pricing pages before relying on them:
- AMP: https://aws.amazon.com/prometheus/pricing/
- AMG: https://aws.amazon.com/grafana/pricing/
- EC2: https://aws.amazon.com/ec2/pricing/on-demand/
- Use the AWS Pricing Calculator for your own estimate.

## 1. What costs money

| Service | What is billed | Reference rate | Notes |
|---|---|---|---|
| **AMP** ingestion | Samples ingested | $0.90 per 10M samples (first 2B/month, cheaper tiers above) | Biggest variable cost. Rate shown is the US reference rate; check the Mumbai rate on the pricing page. |
| **AMP** storage | Compressed samples + metadata | about $0.03 per GB-month | Tiny for this lab. Default retention is 150 days. |
| **AMP** queries | Query samples processed (QSP) | per billion samples (check pricing page) | Dashboards, alert rules and awscurl all count. Dashboards auto-refreshing every few seconds add up. |
| **AMG** | Active users per workspace per month | Editor/Admin $9, Viewer $5 | A workspace needs at least one Editor license. Pricing page lists a free trial for up to 5 users; check current terms. |
| **EC2** | Instance hours | t3.micro Mumbai about $0.0112/hr; t3a.micro about $0.0062/hr | Two instances in this lab. |
| **EBS** | GB-month per volume | small (8 GB root each) | Volumes keep billing if an instance is stopped (not terminated). |
| **Public IPv4** | Per address per hour | about $0.005/hr per address (verify) | Applies to each instance with a public IP, even when idle. |
| **SNS / Identity Center / IAM** | - | free or negligible at this scale | Email notifications are cheap. |
| **CloudWatch** `AWS/Prometheus` metrics | - | no charge for these service metrics | Used only for the ingestion check. |

## 2. Estimate for this lab

Assumptions (change them for your case):
- 2 EC2 instances (target + collector), t3.micro on-demand, ap-south-1
- about 1,600 active series (node_exporter about 1,000 plus a trimmed self-scrape)
- scrape interval 30s = 2,880 samples per series per day

Measure your real series count in Grafana Explore:
```promql
sum by (job) (scrape_samples_post_metric_relabeling)
```

| Item | Math | 3-day lab | Left running 1 month |
|---|---|---|---|
| EC2 (2 x t3.micro) | 2 x $0.0112 x hours | about $1.6 | about $16.4 |
| Public IPv4 (2) | 2 x $0.005 x hours | about $0.7 | about $7.3 |
| EBS (2 x 8 GB) | small | about $0.1 | about $1.5 |
| AMP ingestion | 1,600 x 2,880 = 4.6M samples/day x $0.09 per 1M | about $1.2 | about $12.4 |
| AMP storage + queries | very small | under $0.2 | under $1 |
| AMG (1 editor) | flat per active user | $0 in free trial, otherwise up to $9 | $9 |
| **Total** | | **about $4 (+ AMG)** | **about $45-50** |

With `scrape_interval: 15s` the AMP ingestion line doubles (about $25/month). Using t3a.micro (AMD, about $0.0062/hr) cuts EC2 by roughly 45%.

The lesson: the lab is cheap for a few days and only becomes expensive if it is forgotten. The cleanup step and budget alerts matter more than any tuning.

## 3. Build it cost-aware (design rules used in this repo)
1. **30s scrape interval** (or 60s) instead of 15s: halves ingestion.
2. **Ingest only what you use.** The self-scrape job keeps only `prometheus_remote_storage_*` and `up` (see `configs/prometheus.yml`). Add `metric_relabel_configs` drops for unused node_exporter metrics.
3. **One collector, agent mode, t3.micro or t3a.micro.** No local TSDB, no big disks.
4. **No NAT gateway or interface VPC endpoints** for this lab (they bill hourly). Instances use public IPs and the public AMP endpoint.
5. **Use AMP-managed alert rules** for production-style alerting; AWS recommends native AMP alerting because external systems add extra queries. Keep Grafana alert evaluation intervals at 1m or more.
6. **Dashboard refresh 1m or more**, and short time ranges while testing. Queries are billed by samples processed.
7. **One Editor user only.** Add Viewers only when someone needs them.
8. **Short life.** Set an expiry date tag and delete when done (see cleanup).
9. **Region discipline.** Everything in ap-south-1 so nothing is created (and forgotten) in another region.

## 4. Tagging standard

Apply the same tags to everything so cost and cleanup can be filtered by one key.

| Tag key | Example value | Purpose |
|---|---|---|
| `Project` | `amp-amg-lab` | Main cost filter and cleanup filter |
| `Environment` | `lab` | Separates practice from real workloads |
| `Owner` | `<your-name>` | Who is responsible |
| `Role` | `target`, `collector`, `amp`, `amg`, `alerts` | Cost per component |
| `Expiry` | `2026-10-15` | Delete-by date (YYYY-MM-DD) |

Keep `Monitoring=enabled` and `Name` as they are: `Monitoring` is a **functional tag** used by EC2 service discovery, not a cost tag.

### Where to apply them
- **EC2 launch wizard**: Advanced > Resource tags, and apply to **Instances, Volumes and Network interfaces** so EBS volumes are tagged too.
- **AMP workspace**: Tags section on create (or Tags tab later).
- **AMG workspace**: Tags section on create.
- **SNS topic, IAM role**: tag on create (free, helps cleanup filtering).
- Untagged existing resources: Resource Groups > **Tag Editor** to bulk add tags.

### Activate cost allocation tags (important)
Tags do not show in billing until activated:
1. Billing console > **Cost allocation tags** > select `Project`, `Environment`, `Owner`, `Role` > **Activate**.
2. Wait up to about 24 hours. Data appears only from activation onward, not retroactively.
3. Cost Explorer > Group by **Tag: Project** (or filter `Project = amp-amg-lab`).

Limits to know: auto-assigned public IPv4 charges cannot be tagged and can show up as untagged EC2-Other usage, and tag coverage for AMP/AMG usage lines should be verified in Cost Explorer after activation. Since this lab uses these services only for itself, filtering by **Service** (EC2, Managed Prometheus, Managed Grafana) in Cost Explorer gives the same answer.

## 5. Budget and alerts
Create a small monthly budget so a forgotten resource is caught early:

```bash
./scripts/create-budget.sh you@example.com 10
```
This creates a monthly budget (default USD 10) with email alerts at 50%, 80% and 100% of actual spend. Also consider enabling **Cost Anomaly Detection** (Billing > Cost Anomaly Detection) and using **Budgets** in the console to add a forecasted-spend alert.

Note: Cost data in Billing can lag by several hours to a day, so do not rely on it as the only safeguard. The Expiry tag and cleanup step are the real control.

## 6. After-lab checklist
1. Follow [`setup/06-cleanup.md`](../setup/06-cleanup.md).
2. Run `./scripts/check-leftover-resources.sh` and confirm it lists nothing.
3. Next day, open Cost Explorer (group by Service) and confirm daily cost is about zero.
4. If AMP still shows charges after deleting workspaces, check for workspaces in other regions and see the AMP cost FAQ in the AWS docs.
