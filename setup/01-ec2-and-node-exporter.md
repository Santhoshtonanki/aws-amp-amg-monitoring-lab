# 01 - EC2 instances and node_exporter

Region for the whole lab: **ap-south-1 (Mumbai)**. Select it top-right in the console and keep it the same for every service.

## Launch two instances (EC2 > Launch instance)
| Name | AMI | Type |
|---|---|---|
| target-1 | Amazon Linux 2023 or Ubuntu | t3.micro |
| collector | same AMI | t3.micro |

## Security groups
- **target-sg**: inbound TCP 9100 with source = collector's security group; SSH 22 from My IP.
- **collector-sg**: SSH 22 from My IP only. Do not open 9090 (agent mode has no UI).

## Tags on target-1
EC2 > instance > Tags > Manage tags:
- `Name = target-1`
- `Monitoring = enabled` (the collector auto-discovers targets with this tag)

## Install node_exporter on target-1
Pick the latest version from the node_exporter releases page (use `arm64` on ARM instances).

```bash
NE_VER=<latest-version>
cd /
wget <applicaiton download link address>
tar -xf <applicaiton file name>
ln -s <applicaiton file name> <shortcut name> ## this step will help to make short cut name rather than current file name
cd <shortcut name>  ## there you will find node_exporter files
sudo useradd --no-create-home --shell /bin/false node_exporter ## create user according what you mentioned in "node_exporter.service" file
sudo cp ~/node_exporter.service /etc/systemd/system/node_exporter.service   # from configs/
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s localhost:9100/metrics | head
```

Metrics text in the output means this step is complete. Unit file: [`configs/node_exporter.service`](../configs/node_exporter.service).
