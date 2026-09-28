# 03 - Prometheus agent and remote_write

On the **collector** instance.

## Install
```bash
PROM_VER=<latest-version>
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz
tar xzf prometheus-${PROM_VER}.linux-amd64.tar.gz
sudo cp prometheus-${PROM_VER}.linux-amd64/prometheus /usr/local/bin/
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus/agent
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

## Configure
- Config: [`configs/prometheus.yml`](../configs/prometheus.yml) -> `/etc/prometheus/prometheus.yml` (replace `<WORKSPACE_ID>`)
- Service: [`configs/prometheus.service`](../configs/prometheus.service) -> `/etc/systemd/system/prometheus.service`

Notes:
- `--agent` is for Prometheus 3.x. On 2.x use `--enable-feature=agent` instead.
- Listen address is `127.0.0.1:9090` because agent mode needs no external UI.
- The config includes a **self-scrape job** so Prometheus's own metrics (like `prometheus_remote_storage_*`) are also stored in AMP and can be queried there.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
journalctl -u prometheus -f
```
No 403/400 errors in the logs is a good sign, but logs alone do not prove data is arriving. Verify below.

## Verify before installing Grafana

**1. Remote write counters (on the collector, works without a query API):**
```bash
curl -s localhost:9090/metrics | grep -E 'remote_storage_.*(samples|failed|pending)'
```
Samples sent should increase and failed should stay at 0.

**2. Query AMP itself (use the *query* endpoint, not remote_write):**
```bash
pip install awscurl --break-system-packages
awscurl --service aps --region ap-south-1 \
  "https://aps-workspaces.ap-south-1.amazonaws.com/workspaces/<WORKSPACE_ID>/api/v1/query?query=up" | jq
```
`"status":"success"` with results means ingestion works. Empty result means look at the collector logs, not AMP.

**3. Console check:** AMP workspace > Monitoring tab > query `up`. Optional: CloudWatch > Metrics > AWS/Prometheus > IngestionRate should be non-zero.
