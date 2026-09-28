# 03 - Prometheus agent and remote_write

On the **collector** instance.
## /opt is my optional. you can take your own path: fro example /tmp
## Install
```bash
PROM_VER=<latest-version>
cd /opt
wget <application link address>
tar -xf < tar file name (downloaded one) >
ln -s < file actual name > <shortcut name> ## example:- node_exporter-1.12.1.linux-amd64 as node_exporter (just like a simple name)
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus/agent
sudo chmod 744 /opt/prometheus/
sudo chown -R prometheus:prometheus /var/lib/prometheus
## Note:- 1st refer the Prometheus.service file. give permissions what you are mentioned with in file
```

## Configure
- Config: [`configs/prometheus.yml`](../configs/prometheus.yml) -> `/opt/prometheus/prometheus.yml` (replace `<WORKSPACE_ID>`)
## you can refer the config files for best a best example structure, or else you build you own one by using google or any other AI chat bots
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
    ### example:-
            ### curl -s localhost:9090/metrics | grep -E 'remote_storage_samples
            ### curl -s localhost:9090/metrics | grep -E 'remote_storage_failed
            ### curl -s localhost:9090/metrics | grep -E 'remote_storage_pending
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
