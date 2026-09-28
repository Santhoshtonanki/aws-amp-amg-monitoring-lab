# Issues faced and fixes

Real problems from this lab. Format: **Problem - Cause - Fix - Lesson**.

## 1. Querying AMP with the remote_write URL
- **Problem:** `awscurl` against `.../api/v1/remote_write` failed or returned a meaningless response.
- **Cause:** remote_write is a write-only endpoint used to push data. It cannot be read.
- **Fix:** query with `.../api/v1/query?query=up`.
- **Lesson:** Remote write URL = push metrics into AMP. Query URL = read metrics from AMP.

## 2. Region mismatch
- **Problem:** URLs and commands mixed `us-east-1` and `ap-south-1`.
- **Cause:** the endpoint URL, `sigv4.region`, `awscurl --region` and the workspace region were not the same.
- **Fix:** use one region everywhere (`ap-south-1`) - the workspace, remote_write URL, `sigv4.region`, `awscurl --region` and the Grafana data source default region.
- **Lesson:** a region mismatch shows up as signature or not-found errors that look like permission problems.

## 3. `prometheus_remote_storage_*` metrics missing in queries
- **Problem:** queries for `remote_storage` metrics returned nothing.
- **Cause:** no `job_name: prometheus` self-scrape job. Internal metrics are exposed on `/metrics` but are only stored (and forwarded to AMP) if Prometheus scrapes itself. Also, agent mode has no local query API.
- **Fix:** add the self-scrape job (`localhost:9090`), and check counters directly:
  `curl -s localhost:9090/metrics | grep -E 'remote_storage_.*(samples|failed|pending)'`
- **Lesson:** verify ingestion in two places - collector `/metrics` counters and a query against AMP.

## 4. Logs looked clean but that did not prove ingestion
- **Problem:** no 403/400 in `journalctl`, yet unsure if data reached AMP.
- **Cause:** logs do not show service discovery results or delivered samples.
- **Fix:** check targets are UP, remote write counters increase, then run the `awscurl` query (and the AMP Monitoring tab) for `up`.

## 5. 403 errors from the collector
- **Cause:** IAM role not attached to the instance, or `ec2:DescribeInstances` missing.
- **Fix:** attach `AmazonPrometheusRemoteWriteAccess`, add the `EC2Describe` inline policy, attach the role to the instance, then `sudo systemctl restart prometheus`.

## 6. `awscurl` returns 403
- **Cause:** `AmazonPrometheusQueryAccess` not attached to the role.
- **Fix:** attach it (remove after testing if desired).

## 7. node_exporter service failed to start (permission errors)
- **Problem:** service failed because of ownership/permission problems on the binary path (`/opt/node_exporter/...`) and the user setup.
- **Cause:** the unit ran as a user that could not execute the binary in that location.
- **Fix:** keep the binary in `/usr/local/bin` (root-owned, mode `755`) and run the service as a dedicated `node_exporter` user. Avoid loosening permissions on shared directories such as `/opt` (for example `chmod 766 /opt`), which is a security risk. Then `systemctl daemon-reload && systemctl restart node_exporter` and confirm with `curl localhost:9100/metrics`.
- **Lesson:** fix ownership of the specific binary, not the parent directory.

## 8. systemd unit paths did not match the install
- **Problem:** the unit pointed to `/opt/prometheus/...` while the binary was copied to `/usr/local/bin` and the config lives in `/etc/prometheus`.
- **Fix:** use the same paths in the unit file as in the install steps (see `configs/prometheus.service`).

## 9. Agent flag differs between Prometheus versions
- **Fix:** Prometheus 3.x uses `--agent`; 2.x uses `--enable-feature=agent`. Check the version installed before writing the unit.

## 10. No targets discovered
- **Cause:** missing `Monitoring=enabled` tag, target-sg not allowing 9100 from collector-sg, or missing `ec2:DescribeInstances`.
- **Fix:** check the tag, the security group rule and the IAM policy.

## 11. Grafana data source could not be added
- **Cause:** the Identity Center user had Viewer role.
- **Fix:** AMG workspace > Authentication > Configure users and user groups > **Make admin**.

## 12. Wrong Grafana data source URL
- **Problem:** data source test failed with `/api/v1/query` in the URL.
- **Fix:** use only the workspace base URL `https://aps-workspaces.<region>.amazonaws.com/workspaces/<WORKSPACE_ID>`, SigV4 auth with AWS SDK Default provider, and the correct default region.

## 13. Dashboard 1860 showed "No data"
- **Fix:** match dashboard variables (job/instance) to the labels in use (job `node_exporter`).

## 14. Sensitive values in notes
- **Fix:** before publishing, replaced account ID, workspace IDs, public IPs and the Slack webhook URL with placeholders.
