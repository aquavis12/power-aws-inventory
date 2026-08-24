# AWS Inventory Power

A Kiro Power that scans your AWS account and generates comprehensive infrastructure inventory reports in Excel format — one sheet per service category, fully formatted with filters, color coding, and resource tagging.

Inspired by [aws-auto-inventory](https://github.com/aws-samples/aws-auto-inventory), rebuilt as a Kiro-native agent power with no external tool dependencies.

---

## Features

- **35 service categories** — EC2, S3, RDS, Lambda, VPC, IAM, DynamoDB, EKS, ECS, and 26 more
- **Excel output** — Professional workbook with formatted headers, auto-filters, freeze panes, and alternating row colors
- **Multi-region scanning** — Scan one region or all enabled regions in your account
- **Resource tagging** — Optionally includes tags for every resource (batch-fetched for efficiency)
- **Cost summary** — Optional sheet with 30-day cost breakdown by service
- **Configurable scope** — Pick exactly which services and regions to scan
- **Read-only** — Never creates, modifies, or deletes any AWS resource
- **Error resilient** — Handles access-denied, throttling, and missing services gracefully

---

## Quick Start

1. **Install the power** in Kiro
2. **Ensure AWS credentials** are configured (SSO, env vars, or credentials file)
3. **Say**: "Run an inventory of my AWS account" or "Generate infrastructure report"
4. The agent will:
   - Validate your AWS session
   - Ask which regions/services to scan
   - Discover all resources
   - Generate an Excel workbook at `./inventory-reports/aws-inventory-<accountId>-<date>.xlsx`

---

## Prerequisites

- **AWS credentials** available through the standard credential chain
- **IAM permissions**: Read-only access to the services you want to inventory. A policy like `ReadOnlyAccess` or `ViewOnlyAccess` works well.
- **Python 3.8+** with `openpyxl` installed (for Excel generation):
  ```bash
  pip install openpyxl
  ```

---

## Configuration

Copy `context-templates/inventory-scope.json` to your workspace root and customize:

```json
{
  "regions": ["us-east-1", "eu-west-1"],
  "outputDir": "./inventory-reports",
  "includeTags": true,
  "includeCostSummary": false,
  "categories": ["EC2-Instances", "S3", "Lambda", "RDS"]
}
```

See `context-templates/README.md` for all available options and examples.

---

## Output

The generated Excel workbook includes:

| Sheet | Content |
|-------|---------|
| **Summary** | Account info, scan date, resource counts by category |
| **EC2-Instances** | All EC2 instances with type, state, IPs, VPC, tags |
| **S3** | All buckets with region, versioning, encryption |
| **Lambda** | Functions with runtime, memory, timeout, tags |
| **...** | One sheet per scanned service category |
| **ScanNotes** | Errors, access-denied, throttled calls |

---

## Service Categories (35)

| # | Category | Scope |
|---|----------|-------|
| 1 | IAM Users & Roles | Global |
| 2 | EC2 Instances | Per region |
| 3 | EC2 Security Groups | Per region |
| 4 | VPC & Networking | Per region |
| 5 | S3 Buckets | Global |
| 6 | RDS Instances & Clusters | Per region |
| 7 | DynamoDB Tables | Per region |
| 8 | Lambda Functions | Per region |
| 9 | ECS Clusters & Services | Per region |
| 10 | EKS Clusters | Per region |
| 11 | ElastiCache Clusters | Per region |
| 12 | Redshift Clusters | Per region |
| 13 | OpenSearch Domains | Per region |
| 14 | Load Balancers (ALB/NLB/CLB) | Per region |
| 15 | CloudFront Distributions | Global |
| 16 | Route 53 Hosted Zones | Global |
| 17 | API Gateway | Per region |
| 18 | SQS Queues | Per region |
| 19 | SNS Topics | Per region |
| 20 | Kinesis Streams | Per region |
| 21 | MSK Clusters | Per region |
| 22 | Step Functions | Per region |
| 23 | Glue Jobs & Crawlers | Per region |
| 24 | Athena Workgroups | Per region |
| 25 | EMR Clusters | Per region |
| 26 | SageMaker Endpoints | Per region |
| 27 | KMS Keys | Per region |
| 28 | Secrets Manager | Per region |
| 29 | EBS Volumes | Per region |
| 30 | EFS File Systems | Per region |
| 31 | CloudFormation Stacks | Per region |
| 32 | WAF Web ACLs | Per region + Global |
| 33 | AWS Backup Vaults | Per region |
| 34 | CloudWatch Alarms | Per region |
| 35 | Cost Summary (optional) | Global |

---

## Project Structure

```
power-aws-inventory/
├── context-templates/
│   ├── inventory-scope.json    # Configurable scan scope
│   └── README.md               # Scope configuration docs
├── inventory-reports/          # Output directory (gitignored)
├── scripts/
│   └── generate_excel.py      # Excel workbook generator
├── skills/
│   └── inventory/
│       └── SKILL.md           # Main inventory skill definition
├── steering/
│   ├── inventory-workflow.md  # API calls per service category
│   └── excel-output.md       # Excel formatting rules
├── mcp.json                   # MCP server configuration
├── plugin.json                # Power metadata
├── README.md                  # This file
├── LICENSE                    # MIT License
└── .gitignore
```

---

## MCP Servers Used

| Server | Purpose |
|--------|---------|
| `aws-mcp` | Execute AWS API calls (list/describe operations) |
| `aws-documentation-mcp-server` | Attach relevant AWS docs links |

---

## How It Works

1. **Session validation** — Confirms AWS credentials via `sts:GetCallerIdentity`
2. **Scope configuration** — Loads or prompts for regions and service categories
3. **Resource discovery** — Calls list/describe APIs per service via `aws-mcp`
4. **Tag enrichment** — Batch-fetches tags via Resource Groups Tagging API
5. **Data aggregation** — Collects all results into structured JSON
6. **Excel generation** — Runs `generate_excel.py` to produce the formatted workbook
7. **Summary presentation** — Shows resource counts and file path

---

## Security

- **Read-only**: This power never creates, modifies, or deletes any AWS resource.
- **No secrets in output**: Secret values are never included. Tag values that look like secrets are masked.
- **Local only**: All data stays on your machine. Nothing is uploaded anywhere.

---

## License

MIT — see [LICENSE](./LICENSE)
