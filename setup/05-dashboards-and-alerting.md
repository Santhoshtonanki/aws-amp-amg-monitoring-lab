# 05 - Dashboards and alerting

## A. Import a ready-made dashboard
Dashboards > New > Import > ID `1860` (Node Exporter Full) > Load > select the AMP data source > Import.
If you see "No data", check the dashboard variables (job/instance) match your labels. The job name here is `node_exporter`.

## B. Custom dashboard with a variable
Dashboards > New dashboard > Settings > Variables > New variable
- Name `instance`, Type Query, data source AMP
- Query: `label_values(node_uname_info, instance)`
- Multi-value and Include All enabled

| Panel | Type | PromQL |
|---|---|---|
| CPU % | Time series | `100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle",instance=~"$instance"}[5m])) * 100)` |
| Memory % | Gauge | `(1 - node_memory_MemAvailable_bytes{instance=~"$instance"} / node_memory_MemTotal_bytes{instance=~"$instance"}) * 100` |
| Disk % | Gauge | `100 - (node_filesystem_avail_bytes{mountpoint="/",instance=~"$instance"} / node_filesystem_size_bytes{mountpoint="/",instance=~"$instance"} * 100)` |
| Up/Down | Stat | `up{job="node_exporter",instance=~"$instance"}` |
| Network in | Time series | `rate(node_network_receive_bytes_total{device!="lo",instance=~"$instance"}[5m])` |
| Load | Time series | `node_load1{instance=~"$instance"}` |
| Uptime | Stat | `time() - node_boot_time_seconds{instance=~"$instance"}` |

Save, then Settings > JSON Model > copy it into [`dashboards/ec2-overview.json`](../dashboards/) and commit.

## C. Alerting - Option A: Grafana-managed (Slack)
1. Alerting > Contact points > New: name `slack-alert-contact`, integration Slack, paste the webhook URL, click Test, then Save.
   (Keep the webhook URL out of Git.)
2. Alerting > Alert rules > New alert rule:
   - Name `High CPU Usage Alert`
   - Query A (Code mode): `100 - (avg by(instance_name) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
   - B = Reduce (Last), C = Threshold, is above `80`
   - Folder `demo-alerts`, evaluation group `cpu-checks` every `1m`, pending period `2m`
   - Contact point `slack-alert-contact`
   - Summary: `CPU usage is above 80% on {{ $labels.instance_name }}`

## D. Alerting - Option B: AMP-managed rules + SNS
1. SNS > Create topic (Standard) `amp-alerts`; add an Email subscription and confirm it from the mail.
2. Edit the topic access policy with the statement in [`configs/sns-access-policy-statement.json`](../configs/sns-access-policy-statement.json).
3. AMP > workspace > Rules management > Add namespace > upload [`configs/rules.yaml`](../configs/rules.yaml).
4. AMP > workspace > Alert manager > Add definition > upload [`configs/alertmanager.yaml`](../configs/alertmanager.yaml).

**Test:** on target-1 run `sudo systemctl stop node_exporter`. The `InstanceDown` email arrives in about 2-3 minutes. Start the service again afterwards.
