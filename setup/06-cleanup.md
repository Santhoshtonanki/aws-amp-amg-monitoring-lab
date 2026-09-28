# 06 - Cleanup (avoid bills)

AMG bills per active user and AMP bills for ingestion, so delete everything after practice:
1. Amazon Managed Grafana > workspace > Delete
2. AMP > workspace > Delete
3. EC2 > both instances > Terminate
4. IAM > Roles > `AMPRemoteWriteRole` > Delete
5. SNS > `amp-alerts` topic > Delete (Identity Center user can stay if needed)

Cost tips: use `scrape_interval: 30s` (or 60s) instead of 15s, and keep the lab short-lived.
