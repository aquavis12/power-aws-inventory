---
name: inventory
description: AWS Infrastructure Inventory — discovers all resources across configured regions and services, then generates a comprehensive Excel workbook with one sheet per service category. Supports full scan, category-based scan, quick scan, multi-region, configurable scope, and resource tagging.
---

# AWS Infrastructure Inventory

**When to use**: When the user says "inventory my AWS", "list all resources", "generate infrastructure report", "what's in my account", "export AWS resources to Excel", "full scan", "scan everything", "scan by category", or any request to discover and catalog AWS infrastructure.

---

## Step 1: Validate AWS Session

- Run `aws sts get-caller-identity` via `aws-mcp` to confirm credentials are valid.
- If expired, instruct the user to re-authenticate (`aws login` or `aws sso login`).
- Record the **account ID**, **caller ARN**, and **default region**.
- Display: "Authenticated as `<arn>` in account `<accountId>`"

## Step 2: Load or Configure Inventory Scope

Check if `context-templates/inventory-scope.json` exists in the workspace:

- **If exists**: Load it and display the configured scan mode, regions, and service categories.
- **If not exists**: Ask the user for:
  - **Scan mode** — `full` (all services), `category` (only selected categories), or `quick` (compute, storage, networking only)
  - **Region scope** — `single` (one region), `multi` (list of regions), or `all` (every enabled region)
  - Target region(s) — default: all enabled regions
  - Service categories to scan — default: all (only applies to `category` mode)
  - Output directory — default: `./inventory-reports`
  - Whether to include resource tags — default: yes
  - Whether to include cost data (requires Cost Explorer access) — default: no

Present the scope summary and wait for user confirmation before proceeding.

### Scan Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `full` | Scans ALL service categories (~95 services) across all configured regions | Complete infrastructure audit, compliance review |
| `category` | Scans only the categories listed in the `categories` array | Targeted scan of specific service groups |
| `quick` | Scans only Compute, Storage, and Networking groups | Fast overview of core infrastructure |

### Region Scope

The scan supports **single-region** or **multi-region** modes:

| Region Mode | Description | Use Case |
|-------------|-------------|----------|
| `single` | Scans only ONE specified region (e.g., `us-east-1`) | Fast, focused scan of primary region; dev/test audits |
| `multi` | Scans a specified list of regions (e.g., `["us-east-1", "eu-west-1"]`) | Multi-region workloads, disaster recovery audits |
| `all` | Discovers and scans ALL enabled regions in the account | Full account-wide audit, compliance review |

**Behavior:**
- If `regions` contains a single region → **single-region scan**. Only that region is scanned for per-region services. Global services (IAM, S3, Route53, CloudFront, Organizations, etc.) are always scanned regardless.
- If `regions` contains multiple regions → **multi-region scan**. Each per-region service is scanned in every listed region.
- If `regions` is set to `["all"]` or is empty → **all-region scan**. The agent calls `ec2:DescribeRegions` to discover enabled regions and scans all of them.

**User prompts that trigger each mode:**
- "Scan us-east-1 only" / "Single region scan" → single
- "Scan us-east-1 and eu-west-1" / "Scan these regions: ..." → multi
- "Scan all regions" / "Full scan" / "Scan everything" → all

**Performance considerations:**
- Single-region: fastest (1–3 minutes for full category set)
- Multi-region (2–5 regions): moderate (3–8 minutes)
- All regions (~15+ regions): slowest (10–20 minutes)
- Recommend single or targeted multi-region for iterative work; all-region for compliance audits

### Category-Based Scanning

When `scanMode` is `category`, the user can specify individual services OR entire category groups:

- **By group**: "Scan all Security services" → scans KMS, SecretsManager, WAF, Shield, GuardDuty, SecurityHub, Inspector, Macie, ACM, Firewall-Manager, IAM-AccessAnalyzer
- **By service**: "Scan EC2 and RDS" → scans only EC2-Instances, EC2-SecurityGroups, and RDS
- **Mixed**: "Scan Compute group plus DynamoDB" → scans all Compute services + DynamoDB

## Step 3: Determine Region Scope

Based on the configured `regions` field:

**Single-region** (one region in list):
1. Use that region directly. No discovery needed.
2. Inform the user: "Scanning single region: `<region>`"

**Multi-region** (multiple regions in list):
1. Validate each region is enabled via `ec2:DescribeRegions`.
2. Inform the user: "Scanning N regions: `<region1>, <region2>, ...`"

**All-region** (`["all"]` or empty):
1. Call `ec2:DescribeRegions` via `aws-mcp` to get all enabled regions.
2. Present the list and confirm with the user.
3. For faster scans, recommend starting with the primary region(s) where most resources live.

Global services (IAM, S3, Route53, CloudFront, Organizations, Shield, GlobalAccelerator, TrustedAdvisor, HealthDashboard, Cost Management) are always scanned once regardless of region scope.

## Step 4: Execute Inventory Scan

Follow `steering/inventory-workflow.md` for the detailed API calls per service.

For each service category, for each target region:
1. Call the appropriate `list*` / `describe*` API via `aws-mcp`.
2. Page through ALL results (handle pagination tokens).
3. Collect resource details into structured data.
4. If tags are enabled, fetch tags for each resource (batch where possible via `resourcegroupstaggingapi:GetResources`).
5. Track progress and report to the user: "Scanning [Service] in [Region]... found N resources"

### Service Categories by Group

#### Identity & Access (Global)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 1 | IAM Users, Roles, Policies & Groups | IAM | Global |
| 2 | IAM Identity Center (SSO) | IAM-IdentityCenter | Global |
| 3 | AWS Organizations | Organizations | Global |

#### Compute (Per Region unless noted)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 4 | EC2 Instances | EC2-Instances | Per region |
| 5 | EC2 Security Groups | EC2-SecurityGroups | Per region |
| 6 | EC2 AMIs (owned) | EC2-AMIs | Per region |
| 7 | EC2 Key Pairs | EC2-KeyPairs | Per region |
| 8 | EC2 Placement Groups | EC2-PlacementGroups | Per region |
| 9 | EC2 Launch Templates | EC2-LaunchTemplates | Per region |
| 10 | Lambda Functions | Lambda | Per region |
| 11 | ECS Clusters & Services | ECS | Per region |
| 12 | EKS Clusters | EKS | Per region |
| 13 | Fargate Profiles | Fargate | Per region |
| 14 | Lightsail Instances | Lightsail | Per region |
| 15 | AWS Batch Compute Environments | Batch | Per region |
| 16 | App Runner Services | AppRunner | Per region |
| 17 | Elastic Beanstalk Environments | ElasticBeanstalk | Per region |

#### Storage (Global/Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 18 | S3 Buckets | S3 | Global |
| 19 | EBS Volumes | EBS | Per region |
| 20 | EFS File Systems | EFS | Per region |
| 21 | FSx File Systems | FSx | Per region |
| 22 | S3 Glacier Vaults | S3-Glacier | Per region |
| 23 | Storage Gateway | StorageGateway | Per region |
| 24 | AWS Backup Vaults & Plans | Backup | Per region |

#### Database (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 25 | RDS Instances & Clusters | RDS | Per region |
| 26 | DynamoDB Tables | DynamoDB | Per region |
| 27 | ElastiCache Clusters | ElastiCache | Per region |
| 28 | Redshift Clusters | Redshift | Per region |
| 29 | OpenSearch Domains | OpenSearch | Per region |
| 30 | Neptune Clusters | Neptune | Per region |
| 31 | DocumentDB Clusters | DocumentDB | Per region |
| 32 | Amazon Keyspaces Tables | Keyspaces | Per region |
| 33 | MemoryDB Clusters | MemoryDB | Per region |
| 34 | QLDB Ledgers | QLDB | Per region |
| 35 | Timestream Databases | Timestream | Per region |

#### Networking (Per Region/Global)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 36 | VPC & Subnets & NAT/IGW | VPC | Per region |
| 37 | VPC Endpoints | VPC-Endpoints | Per region |
| 38 | VPC Peering Connections | VPC-PeeringConnections | Per region |
| 39 | Load Balancers (ALB/NLB/CLB/GWLB) | LoadBalancers | Per region |
| 40 | CloudFront Distributions | CloudFront | Global |
| 41 | Route 53 Hosted Zones | Route53 | Global |
| 42 | API Gateway (REST/HTTP/WebSocket) | APIGateway | Per region |
| 43 | Global Accelerator | GlobalAccelerator | Global |
| 44 | Transit Gateways | TransitGateway | Per region |
| 45 | Direct Connect Connections | DirectConnect | Per region |
| 46 | VPN Connections | VPN | Per region |
| 47 | PrivateLink Endpoints | PrivateLink | Per region |
| 48 | Network Firewall | NetworkFirewall | Per region |
| 49 | Elastic IPs | ElasticIP | Per region |

#### Security (Per Region/Global)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 50 | KMS Keys | KMS | Per region |
| 51 | Secrets Manager | SecretsManager | Per region |
| 52 | WAF Web ACLs | WAF | Per region + Global |
| 53 | Shield Protections | Shield | Global |
| 54 | GuardDuty Detectors | GuardDuty | Per region |
| 55 | Security Hub Findings Summary | SecurityHub | Per region |
| 56 | Inspector Findings Summary | Inspector | Per region |
| 57 | Macie Status | Macie | Per region |
| 58 | ACM Certificates | ACM | Per region |
| 59 | Firewall Manager Policies | Firewall-Manager | Per region |
| 60 | IAM Access Analyzer | IAM-AccessAnalyzer | Per region |

#### Application Integration (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 61 | SQS Queues | SQS | Per region |
| 62 | SNS Topics | SNS | Per region |
| 63 | EventBridge Rules & Buses | EventBridge | Per region |
| 64 | Step Functions | StepFunctions | Per region |
| 65 | AppSync APIs | AppSync | Per region |
| 66 | Amazon MQ Brokers | MQ | Per region |
| 67 | SES Identities & Configs | SES | Per region |

#### Analytics (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 68 | Kinesis Streams | Kinesis | Per region |
| 69 | MSK Clusters | MSK | Per region |
| 70 | Glue Jobs, Crawlers & Databases | Glue | Per region |
| 71 | Athena Workgroups | Athena | Per region |
| 72 | EMR Clusters | EMR | Per region |
| 73 | QuickSight Datasets | QuickSight | Per region |
| 74 | Data Pipeline | DataPipeline | Per region |
| 75 | Lake Formation Resources | LakeFormation | Per region |
| 76 | OpenSearch Serverless Collections | OpenSearchServerless | Per region |

#### Machine Learning (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 77 | SageMaker Endpoints & Notebooks | SageMaker | Per region |
| 78 | Bedrock Models & Agents | Bedrock | Per region |
| 79 | Comprehend Endpoints | Comprehend | Per region |
| 80 | Rekognition Collections | Rekognition | Per region |
| 81 | Textract Adapters | Textract | Per region |
| 82 | Transcribe Jobs | Transcribe | Per region |
| 83 | Polly Lexicons | Polly | Per region |
| 84 | Forecast Datasets | Forecast | Per region |

#### Management & Monitoring (Per Region/Global)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 85 | CloudWatch Alarms | CloudWatch | Per region |
| 86 | CloudFormation Stacks | CloudFormation | Per region |
| 87 | CloudTrail Trails | CloudTrail | Per region |
| 88 | AWS Config Rules | Config | Per region |
| 89 | Systems Manager Inventory | SystemsManager | Per region |
| 90 | Trusted Advisor Checks | TrustedAdvisor | Global |
| 91 | Health Dashboard Events | HealthDashboard | Global |
| 92 | Service Catalog Portfolios | ServiceCatalog | Per region |

#### Developer Tools (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 93 | CodeCommit Repositories | CodeCommit | Per region |
| 94 | CodeBuild Projects | CodeBuild | Per region |
| 95 | CodeDeploy Applications | CodeDeploy | Per region |
| 96 | CodePipeline Pipelines | CodePipeline | Per region |
| 97 | CodeArtifact Repositories | CodeArtifact | Per region |
| 98 | Cloud9 Environments | Cloud9 | Per region |
| 99 | ECR Repositories | ECR | Per region |

#### Migration & Transfer (Per Region)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 100 | DMS Replication Instances | DMS | Per region |
| 101 | Transfer Family Servers | TransferFamily | Per region |
| 102 | DataSync Tasks | DataSync | Per region |
| 103 | Migration Hub Applications | MigrationHub | Per region |
| 104 | Snow Family Devices | Snow | Per region |

#### Cost Management (Global)

| # | Category | Sheet Name | Scope |
|---|----------|-----------|-------|
| 105 | Cost Summary (last 30 days) | CostSummary | Global |
| 106 | Budgets | Budgets | Global |
| 107 | Cost Anomaly Detection | CostAnomalyDetection | Global |
| 108 | Savings Plans | SavingsPlans | Global |
| 109 | Reserved Instances | ReservedInstances | Per region |

## Step 5: Handle Errors Gracefully

- If an API call is denied (AccessDenied), log it and continue. Note it in a "ScanNotes" sheet.
- If a service has no resources in a region, skip it (don't create an empty sheet).
- If a region is unreachable, note it and continue with other regions.
- If a service is not available in a region, note as `NOT_AVAILABLE` and continue.
- Retry throttled requests (wait 2–5 seconds, then retry up to 3 times).

## Step 6: Generate Excel Report

Once all data is collected:

1. Write the collected JSON data to a temporary file (`inventory-reports/inventory-data.json`).
2. Run `scripts/generate_excel.py` to produce the Excel workbook.
3. The script creates: `inventory-reports/aws-inventory-<accountId>-<YYYYMMDD-HHMM>.xlsx`
4. Delete the temporary JSON file after successful Excel generation.

Follow `steering/excel-output.md` for formatting rules.

## Step 7: Generate Summary Sheet

The first sheet in the workbook ("Summary") contains:
- Account ID and caller identity
- Scan timestamp
- Scan mode used (full / category / quick)
- Regions scanned
- Total resources found (by category group and individual category)
- Any errors or access-denied notes
- Services with zero resources (skipped)

## Step 8: Present Results

- Print the absolute path to the Excel file.
- Show a brief summary:
  - Scan mode used
  - Total resources discovered across all categories
  - Top 5 categories by resource count
  - Top 5 category groups by resource count
  - Regions scanned
  - Any services that were skipped or had errors
- Offer to open the file or generate additional formats (CSV, JSON).

---

## Full Scan Behavior

When `scanMode` is `full`:
1. All 109 service categories are scanned.
2. Global services (IAM, S3, Route53, CloudFront, Organizations, Shield, GlobalAccelerator, TrustedAdvisor, HealthDashboard, CostSummary, Budgets, CostAnomalyDetection, SavingsPlans) are scanned once.
3. Regional services are scanned in every configured region.
4. Cost Management categories (CostSummary, Budgets, CostAnomalyDetection, SavingsPlans) require Cost Explorer access — skip gracefully if not enabled.
5. Estimated time: 5–15 minutes depending on number of regions and resources.

## Quick Scan Behavior

When `scanMode` is `quick`:
1. Only scans: EC2-Instances, EC2-SecurityGroups, Lambda, ECS, EKS, S3, EBS, EFS, VPC, LoadBalancers, Route53, CloudFront.
2. Provides a fast infrastructure overview in under 2 minutes.
3. Useful for quick audits or initial assessments.

---

## Large Account Strategy — Incremental Multi-Workbook Output

For large AWS accounts (many regions, hundreds or thousands of resources), do NOT attempt to collect all data before generating output. Instead, produce Excel sheets incrementally as each category group completes. This avoids long wait times, memory pressure, and the risk of losing progress if something fails mid-scan.

### Approach: One Workbook Per Category Group + Master Index

1. **Generate a separate Excel workbook per category group** as soon as that group's scan finishes:
   - `inventory-reports/aws-inventory-<accountId>-<date>-Compute.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Storage.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Database.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Networking.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Security.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Analytics.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-AppIntegration.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-ML.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Management.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-DevTools.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Migration.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-CostManagement.xlsx`
   - `inventory-reports/aws-inventory-<accountId>-<date>-Identity.xlsx`

2. **Write each workbook immediately** after its category group scan completes — do NOT wait for the full scan to finish.

3. **Generate a Master Index workbook** at the end:
   - File: `inventory-reports/aws-inventory-<accountId>-<date>-MASTER.xlsx`
   - Contains a "Summary" sheet with account metadata, total resource counts per group, scan timestamp, and regions scanned.
   - Contains a "File Index" sheet listing each group workbook filename, sheet count, and total resources in that workbook.
   - Contains a "ScanNotes" sheet with any errors, access-denied entries, or skipped services.

### When to Use This Strategy

- **Always use for `full` scan mode** — full scans cover 109 service categories and can take 10–20 minutes on large accounts.
- **Use for `category` mode when scanning 3+ category groups** or when multi-region is enabled with 5+ regions.
- **NOT needed for `quick` mode** — quick scans are fast enough to produce a single workbook.

### Benefits

- The user gets partial results within 1–2 minutes instead of waiting 15+ minutes for everything.
- If the scan is interrupted or a category group fails, prior workbooks are already saved.
- Keeps individual workbook file sizes manageable (no 50MB+ single file).
- Easier for the user to share or review specific areas (e.g., just the Security workbook).

### Progress Reporting

After each group workbook is written, inform the user:
```
✓ Compute scan complete — 47 resources across 3 regions → aws-inventory-123456789012-20260825-Compute.xlsx
✓ Storage scan complete — 22 resources across 3 regions → aws-inventory-123456789012-20260825-Storage.xlsx
  ... scanning Database ...
```

### Final Summary

After the Master workbook is generated, present:
- Total workbooks produced
- Total resources across all groups
- Top resource-heavy groups
- Any groups that were skipped or had errors
- Path to the MASTER index workbook

---

## Guardrails

- **Always use the Power**. All AWS interactions MUST go through the `power-aws-inventory` Kiro Power and its MCP servers (`aws-mcp`, `aws-documentation-mcp-server`). NEVER shell out to AWS CLI commands directly (e.g., never run `aws ec2 describe-instances` via terminal). Use the Power's tools exclusively for all API calls, resource discovery, and data retrieval.
- **Read-only**. Never create, modify, or delete any AWS resource.
- **Page through ALL results** — never truncate inventory data.
- **Respect rate limits** — add delays between API calls if throttled.
- **No secrets in output** — never include secret values, passwords, or access keys in the report.
- **Tag values only** — include tag keys and values but mask any tag that looks like a secret (contains "password", "secret", "key", "token" in the name).
- If any API is denied, note it in the ScanNotes sheet and continue.
- Never invent operation names — discover valid operations via `aws-mcp` tools first.
