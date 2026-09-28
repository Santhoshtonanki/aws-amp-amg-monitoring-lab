# 04 - AMG workspace, Identity Center and data source

## A. IAM Identity Center
1. Console > IAM Identity Center > Enable (same region).
2. Users > Add user with a real email; set the password from the invitation mail.

## B. Create the AMG workspace
Console > Amazon Managed Grafana > Create workspace
- Name: `demo-monitoring`
- Authentication: **AWS IAM Identity Center**
- Permission type: **Service managed**
- Outbound VPC connection: skip
    ## if you provide VPC, you need to give subnet information. based on this internet access will be manageable
- Network access: Open access; IP type: IPv4 only
- Encryption: default (AWS managed key)
- Account access: **Current account**
- Data sources: tick **Amazon Managed Service for Prometheus** (CloudWatch optional; extra sources only add unused IAM permissions)
- Notification channels: optional (SNS)

Wait about 5 minutes for **Active**.

## C. Assign the user and make Admin
Workspace > Authentication tab > Configure users and user groups > Assign new user (or group) > select user > **Make it as admin**.

Skipping this still lets you log in, but with Viewer role you cannot access to add data sources.

## D. Log in
Open the workspace URL and sign in with Identity Center.

## E. Add the AMP data source
Connections > Data sources > Amazon Managed Service for Prometheus (or Apps > AWS Data Sources).
- Prometheus server URL: `https://aps-workspaces.ap-south-1.amazonaws.com/workspaces/<WORKSPACE_ID>` (base URL only, no `/api/v1/query`)
- Authentication: **SigV4**, provider **AWS SDK Default**, default region `ap-south-1`
- Assume Role ARN / External ID: leave empty (service-managed)
- Save & test -> success message

Then Explore > select the AMP data source > query `up` > Run query. Series returned means the data source works.
