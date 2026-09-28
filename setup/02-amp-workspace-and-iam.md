# 02 - AMP workspace and IAM role

## Create the AMP workspace
Console > Amazon Managed Service for Prometheus > All workspaces > Create
- Alias: `demo-ec2-monitoring`
- When status is **Active**, note:
  - **Workspace ID**: `ws-xxxxxxxx-....`
  - **Endpoint - remote write URL**: `https://aps-workspaces.ap-south-1.amazonaws.com/workspaces/<WORKSPACE_ID>/api/v1/remote_write`

### Remote write URL vs Query URL
| | Purpose | Path |
|---|---|---|
| Remote write | Push metrics **into** AMP (write-only) | `/api/v1/remote_write` |
| Query | Read metrics **from** AMP | `/api/v1/query` |

Grafana uses the workspace base URL (no `/api/v1/...`) and appends API paths itself.

IPv4 vs dual-stack: the lab uses the IPv4 endpoint. Dual-stack (`.api.aws`) is only needed if IPv6 is properly configured.

## IAM role for the collector
IAM > Roles > Create role
- Trusted entity: AWS service > EC2
- Policies: `AmazonPrometheusRemoteWriteAccess`, and `AmazonPrometheusQueryAccess` (used for the `awscurl` test; can be removed later)
- Role name: `AMPRemoteWriteRole`

Creating the role in the console also creates the instance profile automatically.

### Inline policy for EC2 discovery
Role > Add permissions > Create inline policy > JSON, name `EC2Describe`:
see [`configs/iam-ec2-describe-policy.json`](../configs/iam-ec2-describe-policy.json).

### Attach to the collector
EC2 > collector > Actions > Security > Modify IAM role > `AMPRemoteWriteRole` > Update.
