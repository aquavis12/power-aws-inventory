---
name: inventory
description: AWS Infrastructure Inventory — discovers all resources across configured regions and services, then generates a comprehensive Excel workbook with one sheet per service category. Supports full scan, category-based scan, quick scan, multi-region, configurable scope, and resource tagging.
---

# AWS Infrastructure Inventory

**When to use**: When the user says "inventory my AWS", "list all resources", "generate infrastructure report", "what's in my account", "export AWS resources to Excel", "full scan", "scan everything", "scan by category", or any request to discover and catalog AWS infrastructure.

---

## Step 1: Validate AWS Session

1. Run `GetCallerIdentity` via `aws-mcp` `run_script` to confirm credentials.
2. Record **account ID**, **caller ARN**, and **default region**.
3. Display: "Authenticated as `<arn>` in account `<accountId>`"

**Profile handling**: The `aws-mcp` sandbox uses the **default** AWS profile only — it cannot use `--profile` or set environment variables. If the default profile points to the wrong account:
- Check profiles via CLI: `aws configure list-profiles`
- Verify target: `aws sts get-caller-identity --profile <name>`
- Instruct user to update their default profile or set `AWS_PROFILE` env var before proceeding.

If expired: instruct user to re-authenticate (`aws sso login` or equivalent).

## Step 2: Configure Scope & Regions

Load `context-templates/inventory-scope.json` if it exists; otherwise ask the user:

| Setting | Options | Default |
|---------|---------|---------|
| Scan mode | `full` / `category` / `quick` | full |
| Region scope | `single` / `multi` / `all` | all |
| Include tags | yes / no | yes |
| Include cost data | yes / no (requires Cost Explorer) | no |
| Output dir | path | `./inventory-reports` |

**Region resolution:**
- `single` → use specified region directly
- `multi` → validate via `DescribeRegions`, use listed regions
- `all` → call `DescribeRegions` to discover all enabled regions, then **exclude** `me-south-1` and `me-central-1` (Bahrain/Dubai — on outage, do not scan).

Global services (IAM, S3, Route53, CloudFront, Organizations, Shield, GlobalAccelerator, TrustedAdvisor, HealthDashboard, Cost Management) always scan once regardless of region scope.

**Category-based scanning** (`category` mode): User can specify groups ("Scan all Security services"), individual services ("Scan EC2 and RDS"), or mixed.

Present scope summary and confirm before proceeding.

## Step 3: Execute Inventory Scan

Refer to `steering/inventory-workflow.md` for per-service API calls.

**Execution approach — use `run_script` with `asyncio.gather`:**
- Scan all regions in parallel per service (far faster than sequential CLI calls).
- Batch regions into groups of 10–12 per `run_script` call to avoid timeouts.
- `run_script` auto-paginates — no manual token handling needed.

**Per-service workflow:**
1. Call the appropriate `List*` / `Describe*` API for each region.
2. Collect resource details into structured data.
3. If tags enabled, batch-fetch via `resourcegroupstaggingapi:GetResources`.
4. Report progress: "Scanning [Service] in [Region]... found N resources"

### Critical Implementation Notes

| Service | Gotcha | Solution |
|---------|--------|----------|
| **S3** | `ListBuckets` has no region | Call `GetBucketLocation` per bucket. `null` = `us-east-1`. Batch in groups of 20–30 to avoid throttling. |
| **ECS** | `ListClusters` returns ARNs only (key: `clusterArns`) | Must follow with `DescribeClusters(clusters=<arns>)` for details. |
| **EBS** | Attachment is nested | Access `Attachments[0].InstanceId`. State `available` = orphaned. |
| **EC2** | Instances nested in Reservations | Flatten: `Reservations[].Instances[]`. Get Name from `Tags[?Key=='Name'].Value`. |
| **DynamoDB** | `ListTables` returns names only | Follow with `DescribeTable` per table for item count/size. |
| **EKS** | `ListClusters` returns names | Follow with `DescribeCluster` per name for version/status/endpoint. |

### Handling Timeouts & Errors

- If `run_script` returns `{task_id, status: "working"}` → it exceeded execution time. Split into smaller batches.
- **Excluded regions**: `me-south-1` (Bahrain) and `me-central-1` (UAE/Dubai) — skip these entirely. They are on outage and should not be scanned.
- Glue `GetDatabases` can timeout in some regions. Note and continue.
- Throttling: power auto-retries; no manual handling needed.

### Service Categories (109 total)

#### Global Services (scanned once)

| # | Category | Sheet Name |
|---|----------|-----------|
| 1 | IAM Users, Roles, Policies & Groups | IAM |
| 2 | IAM Identity Center (SSO) | IAM-IdentityCenter |
| 3 | AWS Organizations | Organizations |
| 18 | S3 Buckets | S3 |
| 40 | CloudFront Distributions | CloudFront |
| 41 | Route 53 Hosted Zones | Route53 |
| 43 | Global Accelerator | GlobalAccelerator |
| 53 | Shield Protections | Shield |
| 90 | Trusted Advisor Checks | TrustedAdvisor |
| 91 | Health Dashboard Events | HealthDashboard |
| 105–108 | Cost Management (Summary, Budgets, Anomaly, Savings Plans) | CostSummary / Budgets / CostAnomalyDetection / SavingsPlans |

#### Per-Region Services

**Compute:** EC2-Instances, EC2-SecurityGroups, EC2-AMIs, EC2-KeyPairs, EC2-PlacementGroups, EC2-LaunchTemplates, Lambda, ECS, EKS, Fargate, Lightsail, Batch, AppRunner, ElasticBeanstalk

**Storage:** EBS, EFS, FSx, S3-Glacier, StorageGateway, Backup

**Database:** RDS, DynamoDB, ElastiCache, Redshift, OpenSearch, Neptune, DocumentDB, Keyspaces, MemoryDB, QLDB, Timestream

**Networking:** VPC, VPC-Endpoints, VPC-PeeringConnections, LoadBalancers, APIGateway, TransitGateway, DirectConnect, VPN, PrivateLink, NetworkFirewall, ElasticIP

**Security:** KMS, SecretsManager, WAF (Regional + Global), GuardDuty, SecurityHub, Inspector, Macie, ACM, Firewall-Manager, IAM-AccessAnalyzer

**Application Integration:** SQS, SNS, EventBridge, StepFunctions, AppSync, MQ, SES

**Analytics:** Kinesis, MSK, Glue, Athena, EMR, QuickSight, DataPipeline, LakeFormation, OpenSearchServerless

**Machine Learning:** SageMaker, Bedrock, Comprehend, Rekognition, Textract, Transcribe, Polly, Forecast

**Management & Monitoring:** CloudWatch, CloudFormation, CloudTrail, Config, SystemsManager, ServiceCatalog

**Developer Tools:** CodeCommit, CodeBuild, CodeDeploy, CodePipeline, CodeArtifact, Cloud9, ECR

**Migration & Transfer:** DMS, TransferFamily, DataSync, MigrationHub, Snow

**Cost (Per Region):** ReservedInstances

## Step 4: Handle Errors

- **AccessDenied** → log in ScanNotes, continue.
- **No resources** → skip (don't create empty sheet).
- **Region unreachable / timeout** → note and continue with other regions.
- **Service not available in region** → note as `NOT_AVAILABLE`, continue.
- **Throttled** → wait 2–5s, retry up to 3 times.

## Step 5: Generate Excel Report

1. Write JSON data to `inventory-reports/inventory-data.json`.
2. Run `scripts/generate_excel.py` to produce the workbook.
3. Output: `inventory-reports/aws-inventory-<accountId>-<YYYYMMDD-HHMM>.xlsx`
4. Delete temp JSON after success.

**File locking**: If file is open (PermissionError), write to a **new filename** — never block.

**Completeness**: Every sheet must contain ALL discovered resources. Never truncate or sample.

Follow `steering/excel-output.md` for formatting rules.

### Workbook Structure

1. **Summary** (first sheet) — account ID, caller, scan date, mode, regions, total counts by category
2. **One sheet per category** — only for categories with resources
3. **ScanNotes** (last sheet) — errors, access-denied, timeouts, skipped services

## Step 6: Present Results

Show:
- Path to Excel file
- Total resources discovered
- Top 5 categories by count
- Regions scanned
- Any errors or skipped services
- Offer CSV/JSON export if requested

---

## Scan Mode Quick Reference

| Mode | Categories Scanned | Estimated Time |
|------|-------------------|----------------|
| `full` | All 109 | 5–15 min (all regions) |
| `category` | User-specified groups/services | Varies |
| `quick` | EC2, SGs, Lambda, ECS, EKS, S3, EBS, EFS, VPC, LBs, Route53, CloudFront | < 2 min |

---

## Large Account Strategy — Incremental Output

For large accounts (many regions, 500+ resources), produce workbooks incrementally:

1. **One workbook per category group** as each finishes (e.g., `*-Compute.xlsx`, `*-Storage.xlsx`).
2. **Write immediately** — don't wait for the full scan.
3. **Generate MASTER workbook** at the end containing ALL sheets from every group, plus Summary and ScanNotes.

Report progress after each group:
```
✓ Compute — 47 resources → aws-inventory-...-Compute.xlsx
✓ Storage — 22 resources → aws-inventory-...-Storage.xlsx
  ... scanning Database ...
```

Use this strategy for `full` mode or `category` mode with 3+ groups. Not needed for `quick` mode.

---

## Guardrails

- **Always use the Power**. All AWS calls MUST go through `aws-mcp` `run_script` or `call_aws`. NEVER shell out to AWS CLI directly via terminal.
- **Read-only**. Never create, modify, or delete any AWS resource.
- **Complete pagination** — never truncate inventory data.
- **No secrets in output** — mask tags whose key contains "password", "secret", "key", "token", "credential".
- **AccessDenied → ScanNotes** — log and continue, never fail the entire scan.
- **Never invent API operations** — use only documented operations from `steering/inventory-workflow.md`.

### Sensitive Data Handling

This power reads AWS account metadata that may be sensitive. Handle it as follows:

- **Lambda / ECS / Batch environment variables**: Record only the variable **names** (keys), never their **values**. Do NOT call `lambda:GetFunctionConfiguration` for the purpose of exporting `Environment.Variables` values — capture the key list only, or skip entirely.
- **IAM policy documents**: Capture policy **names, ARNs, and attachment counts** only. Do NOT embed full inline or managed policy JSON documents in the workbook.
- **Secrets Manager / SSM SecureString**: Never call `GetSecretValue` or `GetParameter` with decryption. Metadata only (name, ARN, rotation status).
- **Resource identifiers** (ARNs, IPs, account IDs, endpoints): These are included by design for inventory purposes. Treat the generated workbook as sensitive — it is written locally only and never uploaded anywhere.
- **Output stays local**: All data remains in `inventory-reports/` on the user's machine. Never transmit inventory data to any external endpoint.
- The MCP proxy runs with `--read-only`, so write-capable AWS tools are hidden at the transport layer — but these content rules still apply to the read data that is collected.
