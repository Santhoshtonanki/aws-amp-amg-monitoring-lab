# Architecture

```mermaid
flowchart LR
  subgraph VPC[Default VPC - ap-south-1]
    T[target-1<br/>node_exporter :9100<br/>tag Monitoring=enabled]
    C[collector<br/>Prometheus agent<br/>IAM role AMPRemoteWriteRole]
    C -->|scrape :9100| T
  end
  C -->|remote_write SigV4| AMP[(AMP workspace)]
  AMG[AMG workspace] -->|SigV4 query| AMP
  U[User via IAM Identity Center] --> AMG
  AMG -->|alert| SL[Slack]
  AMP -->|Alertmanager| SNS[SNS -> Email]
```

## Data flow
1. `node_exporter` exposes host metrics on port 9100 of the target.
2. The collector discovers targets through the EC2 API (tag filter) and scrapes them every 30s.
3. Samples are pushed to AMP with SigV4-signed `remote_write`, using the collector's instance role.
4. AMG reads from AMP using its service-managed role; users sign in through IAM Identity Center.

## Security groups
| SG | Inbound |
|---|---|
| target-sg | TCP 9100 from collector-sg; SSH 22 from my IP |
| collector-sg | SSH 22 from my IP only (agent mode has no UI, 9090 stays closed) |
