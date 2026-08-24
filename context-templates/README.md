# Context Templates

This folder contains configuration templates for the AWS Inventory Power.

## inventory-scope.json

Controls which regions and services are scanned during an inventory run.

### Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `regions` | string[] | `["us-east-1"]` | AWS regions to scan. Use `["all"]` to scan all enabled regions. |
| `outputDir` | string | `./inventory-reports` | Directory where the Excel report is saved. |
| `includeTags` | boolean | `true` | Whether to fetch resource tags (adds scan time but provides richer data). |
| `includeCostSummary` | boolean | `false` | Whether to include a cost breakdown sheet (requires Cost Explorer access). |
| `categories` | string[] | all 35 categories | Which service categories to scan. Remove entries to skip them. |

### Usage

1. Copy this file to your workspace root as `inventory-scope.json`.
2. Edit the regions and categories to match your needs.
3. Run the inventory skill — it will pick up the scope file automatically.

### Examples

**Scan all regions, all services:**
```json
{
  "regions": ["all"],
  "outputDir": "./inventory-reports",
  "includeTags": true,
  "includeCostSummary": true,
  "categories": ["all"]
}
```

**Scan only compute resources in us-east-1 and eu-west-1:**
```json
{
  "regions": ["us-east-1", "eu-west-1"],
  "outputDir": "./inventory-reports",
  "includeTags": true,
  "includeCostSummary": false,
  "categories": [
    "EC2-Instances",
    "EC2-SecurityGroups",
    "Lambda",
    "ECS",
    "EKS",
    "EMR",
    "SageMaker"
  ]
}
```

**Quick networking audit:**
```json
{
  "regions": ["us-east-1"],
  "outputDir": "./inventory-reports",
  "includeTags": false,
  "includeCostSummary": false,
  "categories": [
    "VPC",
    "EC2-SecurityGroups",
    "LoadBalancers",
    "CloudFront",
    "Route53",
    "WAF"
  ]
}
```

### Available Categories

| Category Key | Description |
|-------------|-------------|
| `IAM` | IAM Users, Roles, Policies, Groups |
| `EC2-Instances` | EC2 Instances |
| `EC2-SecurityGroups` | Security Groups |
| `VPC` | VPCs, Subnets, NAT/IGW/TGW |
| `S3` | S3 Buckets |
| `RDS` | RDS Instances & Aurora Clusters |
| `DynamoDB` | DynamoDB Tables |
| `Lambda` | Lambda Functions |
| `ECS` | ECS Clusters & Services |
| `EKS` | EKS Clusters |
| `ElastiCache` | ElastiCache Clusters |
| `Redshift` | Redshift Clusters |
| `OpenSearch` | OpenSearch Domains |
| `LoadBalancers` | ALB, NLB, CLB, GWLB |
| `CloudFront` | CloudFront Distributions |
| `Route53` | Route 53 Hosted Zones |
| `APIGateway` | REST, HTTP, WebSocket APIs |
| `SQS` | SQS Queues |
| `SNS` | SNS Topics |
| `Kinesis` | Kinesis Data Streams |
| `MSK` | Managed Streaming for Kafka |
| `StepFunctions` | Step Functions State Machines |
| `Glue` | Glue Jobs, Crawlers, Databases |
| `Athena` | Athena Workgroups |
| `EMR` | EMR Clusters |
| `SageMaker` | SageMaker Endpoints & Notebooks |
| `KMS` | KMS Keys |
| `SecretsManager` | Secrets Manager Secrets |
| `EBS` | EBS Volumes |
| `EFS` | EFS File Systems |
| `CloudFormation` | CloudFormation Stacks |
| `WAF` | WAF Web ACLs |
| `Backup` | AWS Backup Vaults & Plans |
| `CloudWatch` | CloudWatch Alarms |
| `CostSummary` | Cost breakdown by service (optional) |
