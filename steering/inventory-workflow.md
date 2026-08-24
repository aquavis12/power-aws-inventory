# Inventory Workflow — Service Categories & API Calls

This document defines the exact AWS API calls to make for each service category. All calls are **read-only**. Use `aws-mcp` to execute each call.

---

## Pagination Rules

- Always handle pagination. Look for `NextToken`, `Marker`, `NextMarker`, or `IsTruncated` in responses.
- Continue calling until no more pages remain.
- Never truncate results — the inventory must be complete.

## Rate Limiting

- If you receive a `Throttling` or `TooManyRequestsException`, wait 3 seconds and retry (max 3 retries).
- Space regional scans 1 second apart to avoid account-level throttling.

## Tag Fetching Strategy

For services that don't return tags inline, use one of:
1. **Batch tagging**: `resourcegroupstaggingapi:GetResources` with resource type filter — most efficient.
2. **Per-resource**: `<service>:ListTagsForResource` with the resource ARN — fallback.

Prefer batch tagging when scanning more than 5 resources of the same type.

## Scan Mode Handling

- **full**: Execute ALL categories below (1–109).
- **category**: Execute only the categories listed in the scope file's `categories` array.
- **quick**: Execute only categories marked with ⚡ (quick scan eligible).

---

# Group 1: Identity & Access (Global)

---

## Category 1: IAM Users & Roles ⚡

**Sheet**: `IAM`

### API Calls:
```
iam:ListUsers → collect UserName, UserId, Arn, CreateDate, PasswordLastUsed
iam:ListRoles → collect RoleName, RoleId, Arn, CreateDate, MaxSessionDuration
iam:ListPolicies (Scope=Local) → collect PolicyName, Arn, AttachmentCount, CreateDate
iam:ListGroups → collect GroupName, GroupId, Arn, CreateDate
```

### Columns:
| Type | Name | ID | ARN | Created | Last Used / Attachment Count | Path |

---

## Category 2: IAM Identity Center (SSO)

**Sheet**: `IAM-IdentityCenter`

### API Calls:
```
sso-admin:ListInstances → get SSO instance ARN
sso-admin:ListPermissionSets (InstanceArn=<arn>) → permission set ARNs
For each permission set:
  sso-admin:DescribePermissionSet (InstanceArn=<arn>, PermissionSetArn=<ps_arn>) → details
identitystore:ListUsers (IdentityStoreId=<id>) → SSO users
identitystore:ListGroups (IdentityStoreId=<id>) → SSO groups
```

### Columns:
| Resource Type | Name | ID/ARN | Description | Session Duration | Created |

---

## Category 3: AWS Organizations

**Sheet**: `Organizations`

### API Calls:
```
organizations:DescribeOrganization → org details
organizations:ListAccounts → all accounts
organizations:ListRoots → root info
organizations:ListOrganizationalUnitsForParent (ParentId=<root>) → OUs (recursive)
```

### Columns:
| Resource Type | Name | ID | Email | Status | Joined Method | Joined Date |

---

# Group 2: Compute (Per Region)

---

## Category 4: EC2 Instances ⚡

**Sheet**: `EC2-Instances`

### API Calls:
```
ec2:DescribeInstances → collect all reservations/instances
```

### Columns:
| Region | Instance ID | Name (tag) | Type | State | VPC ID | Subnet ID | Private IP | Public IP | AMI ID | Launch Time | Platform | Tags |

---

## Category 5: EC2 Security Groups ⚡

**Sheet**: `EC2-SecurityGroups`

### API Calls:
```
ec2:DescribeSecurityGroups → collect all security groups
```

### Columns:
| Region | Group ID | Group Name | VPC ID | Description | Inbound Rules Count | Outbound Rules Count | Tags |

---

## Category 6: EC2 AMIs (Owned)

**Sheet**: `EC2-AMIs`

### API Calls:
```
ec2:DescribeImages (Owners=self) → owned AMIs
```

### Columns:
| Region | Image ID | Name | State | Architecture | Platform | Creation Date | Public | Tags |

---

## Category 7: EC2 Key Pairs

**Sheet**: `EC2-KeyPairs`

### API Calls:
```
ec2:DescribeKeyPairs → all key pairs
```

### Columns:
| Region | Key Pair ID | Name | Type | Fingerprint | Created | Tags |

---

## Category 8: EC2 Placement Groups

**Sheet**: `EC2-PlacementGroups`

### API Calls:
```
ec2:DescribePlacementGroups → all placement groups
```

### Columns:
| Region | Group Name | Group ID | Strategy | State | Partition Count | Tags |

---

## Category 9: EC2 Launch Templates

**Sheet**: `EC2-LaunchTemplates`

### API Calls:
```
ec2:DescribeLaunchTemplates → all launch templates
```

### Columns:
| Region | Template ID | Name | Default Version | Latest Version | Created By | Created | Tags |

---

## Category 10: Lambda Functions ⚡

**Sheet**: `Lambda`

### API Calls:
```
lambda:ListFunctions → all functions with details
```

### Columns:
| Region | Function Name | Runtime | Memory (MB) | Timeout (s) | Code Size | Handler | Last Modified | Architecture | Tags |

---

## Category 11: ECS Clusters & Services ⚡

**Sheet**: `ECS`

### API Calls:
```
ecs:ListClusters → cluster ARNs
ecs:DescribeClusters (clusters=<arns>) → cluster details
For each cluster:
  ecs:ListServices (cluster=<arn>) → service ARNs
  ecs:DescribeServices (cluster=<arn>, services=<arns>) → service details
```

### Columns:
| Region | Cluster Name | Cluster Status | Active Services | Running Tasks | Pending Tasks | Service Name | Service Status | Desired Count | Running Count | Launch Type | Tags |

---

## Category 12: EKS Clusters ⚡

**Sheet**: `EKS`

### API Calls:
```
eks:ListClusters → cluster names
For each cluster:
  eks:DescribeCluster (name=<cluster>) → details
```

### Columns:
| Region | Cluster Name | Version | Status | Platform Version | Endpoint | VPC ID | Subnet IDs | Created | Tags |

---

## Category 13: Fargate Profiles

**Sheet**: `Fargate`

### API Calls:
```
eks:ListClusters → cluster names
For each cluster:
  eks:ListFargateProfiles (clusterName=<name>) → profile names
  For each profile:
    eks:DescribeFargateProfile (clusterName=<name>, fargateProfileName=<profile>) → details
```

### Columns:
| Region | Cluster Name | Profile Name | Status | Pod Execution Role | Subnets | Selectors | Created | Tags |

---

## Category 14: Lightsail Instances

**Sheet**: `Lightsail`

### API Calls:
```
lightsail:GetInstances → all instances
lightsail:GetDatabases → all databases
lightsail:GetLoadBalancers → all load balancers
```

### Columns:
| Region | Resource Type | Name | State | Blueprint | Bundle | Public IP | Created | Tags |

---

## Category 15: AWS Batch

**Sheet**: `Batch`

### API Calls:
```
batch:DescribeComputeEnvironments → compute environments
batch:DescribeJobQueues → job queues
```

### Columns:
| Region | Resource Type | Name | State | Status | Type | vCPUs (min/max/desired) | Tags |

---

## Category 16: App Runner Services

**Sheet**: `AppRunner`

### API Calls:
```
apprunner:ListServices → service summaries
For each service:
  apprunner:DescribeService (ServiceArn=<arn>) → details
```

### Columns:
| Region | Service Name | Service ARN | Status | Source Type | URL | CPU | Memory | Created | Tags |

---

## Category 17: Elastic Beanstalk

**Sheet**: `ElasticBeanstalk`

### API Calls:
```
elasticbeanstalk:DescribeApplications → all apps
elasticbeanstalk:DescribeEnvironments → all environments
```

### Columns:
| Region | Resource Type | App/Env Name | Status | Platform | Solution Stack | URL | Updated | Tags |

---

# Group 3: Storage (Global/Per Region)

---

## Category 18: S3 Buckets ⚡ (Global)

**Sheet**: `S3`

### API Calls:
```
s3:ListBuckets → bucket names and creation dates
For each bucket:
  s3:GetBucketLocation → region
  s3:GetBucketTagging → tags (handle NoSuchTagSet gracefully)
  s3:GetBucketVersioning → versioning status
  s3:GetBucketEncryption → encryption config (handle ServerSideEncryptionConfigurationNotFoundError)
```

### Columns:
| Bucket Name | Region | Created | Versioning | Encryption | Tags |

---

## Category 19: EBS Volumes ⚡

**Sheet**: `EBS`

### API Calls:
```
ec2:DescribeVolumes → all volumes
```

### Columns:
| Region | Volume ID | Name (tag) | Type | Size (GB) | IOPS | State | Encrypted | Attached To | AZ | Created | Tags |

---

## Category 20: EFS File Systems ⚡

**Sheet**: `EFS`

### API Calls:
```
efs:DescribeFileSystems → all file systems
```

### Columns:
| Region | File System ID | Name | Size (Bytes) | Performance Mode | Throughput Mode | Encrypted | State | Created | Tags |

---

## Category 21: FSx File Systems

**Sheet**: `FSx`

### API Calls:
```
fsx:DescribeFileSystems → all FSx file systems
```

### Columns:
| Region | File System ID | Name | Type (Lustre/Windows/ONTAP/OpenZFS) | Storage (GB) | Lifecycle | VPC ID | Created | Tags |

---

## Category 22: S3 Glacier Vaults

**Sheet**: `S3-Glacier`

### API Calls:
```
glacier:ListVaults → all vaults
```

### Columns:
| Region | Vault Name | ARN | Archives Count | Size (Bytes) | Last Inventory | Created |

---

## Category 23: Storage Gateway

**Sheet**: `StorageGateway`

### API Calls:
```
storagegateway:ListGateways → all gateways
For each gateway:
  storagegateway:DescribeGatewayInformation (GatewayARN=<arn>) → details
```

### Columns:
| Region | Gateway Name | Gateway ID | Type | State | Host Environment | Last Update | Tags |

---

## Category 24: AWS Backup Vaults & Plans

**Sheet**: `Backup`

### API Calls:
```
backup:ListBackupVaults → vault list
backup:ListBackupPlans → plan list
```

### Columns:
| Region | Resource Type (Vault/Plan) | Name | ARN | Recovery Points | Status | Created | Tags |

---

# Group 4: Database (Per Region)

---

## Category 25: RDS Instances & Clusters

**Sheet**: `RDS`

### API Calls:
```
rds:DescribeDBInstances → all DB instances
rds:DescribeDBClusters → all Aurora clusters
```

### Columns:
| Region | Type (Instance/Cluster) | Identifier | Engine | Engine Version | Class | Status | Multi-AZ | Storage (GB) | Endpoint | Created | Tags |

---

## Category 26: DynamoDB Tables

**Sheet**: `DynamoDB`

### API Calls:
```
dynamodb:ListTables → table names
For each table:
  dynamodb:DescribeTable → details
```

### Columns:
| Region | Table Name | Status | Item Count | Size (Bytes) | Billing Mode | Read Capacity | Write Capacity | Created | Tags |

---

## Category 27: ElastiCache Clusters

**Sheet**: `ElastiCache`

### API Calls:
```
elasticache:DescribeCacheClusters (ShowCacheNodeInfo=true) → cluster details
elasticache:DescribeReplicationGroups → replication groups
```

### Columns:
| Region | Cluster ID | Engine | Engine Version | Node Type | Num Nodes | Status | Replication Group | Created | Tags |

---

## Category 28: Redshift Clusters

**Sheet**: `Redshift`

### API Calls:
```
redshift:DescribeClusters → all clusters
```

### Columns:
| Region | Cluster ID | Node Type | Num Nodes | Status | DB Name | Endpoint | Encrypted | Created | Tags |

---

## Category 29: OpenSearch Domains

**Sheet**: `OpenSearch`

### API Calls:
```
opensearch:ListDomainNames → domain names
For each domain:
  opensearch:DescribeDomain (DomainName=<name>) → details
```

### Columns:
| Region | Domain Name | Engine Version | Instance Type | Instance Count | Status | Endpoint | Encrypted | Created | Tags |

---

## Category 30: Neptune Clusters

**Sheet**: `Neptune`

### API Calls:
```
neptune:DescribeDBClusters → all Neptune clusters
neptune:DescribeDBInstances (Filters=[{Name=engine,Values=[neptune]}]) → Neptune instances
```

### Columns:
| Region | Type (Cluster/Instance) | Identifier | Engine Version | Class | Status | Multi-AZ | Storage Encrypted | Endpoint | Created | Tags |

---

## Category 31: DocumentDB Clusters

**Sheet**: `DocumentDB`

### API Calls:
```
docdb:DescribeDBClusters (Filters=[{Name=engine,Values=[docdb]}]) → DocumentDB clusters
docdb:DescribeDBInstances (Filters=[{Name=engine,Values=[docdb]}]) → DocumentDB instances
```

### Columns:
| Region | Type (Cluster/Instance) | Identifier | Engine Version | Class | Status | Multi-AZ | Storage Encrypted | Endpoint | Created | Tags |

---

## Category 32: Amazon Keyspaces (Cassandra)

**Sheet**: `Keyspaces`

### API Calls:
```
keyspaces:ListKeyspaces → keyspace names
For each keyspace:
  keyspaces:ListTables (keyspaceName=<name>) → table names
```

### Columns:
| Region | Keyspace Name | Table Name | Status | Tags |

---

## Category 33: MemoryDB Clusters

**Sheet**: `MemoryDB`

### API Calls:
```
memorydb:DescribeClusters → all clusters
```

### Columns:
| Region | Cluster Name | Node Type | Num Shards | Status | Engine Version | Encryption | Endpoint | Created | Tags |

---

## Category 34: QLDB Ledgers

**Sheet**: `QLDB`

### API Calls:
```
qldb:ListLedgers → all ledgers
For each ledger:
  qldb:DescribeLedger (Name=<name>) → details
```

### Columns:
| Region | Ledger Name | State | Permissions Mode | Deletion Protection | Encryption | Created | Tags |

---

## Category 35: Timestream Databases

**Sheet**: `Timestream`

### API Calls:
```
timestream-write:ListDatabases → database list
For each database:
  timestream-write:ListTables (DatabaseName=<name>) → tables
```

### Columns:
| Region | Database Name | Table Name | Status | Retention (Memory/Magnetic) | Created | Tags |

---

# Group 5: Networking (Per Region/Global)

---

## Category 36: VPC & Networking ⚡

**Sheet**: `VPC`

### API Calls:
```
ec2:DescribeVpcs → VPC details
ec2:DescribeSubnets → Subnet details
ec2:DescribeNatGateways → NAT Gateways
ec2:DescribeInternetGateways → IGWs
ec2:DescribeVpnGateways → VPN Gateways
```

### Columns:
| Region | Resource Type | ID | Name (tag) | CIDR / Details | State | Attached To | Tags |

---

## Category 37: VPC Endpoints

**Sheet**: `VPC-Endpoints`

### API Calls:
```
ec2:DescribeVpcEndpoints → all VPC endpoints
```

### Columns:
| Region | Endpoint ID | VPC ID | Service Name | Type (Interface/Gateway) | State | Route Tables / Subnets | Created | Tags |

---

## Category 38: VPC Peering Connections

**Sheet**: `VPC-PeeringConnections`

### API Calls:
```
ec2:DescribeVpcPeeringConnections → all peering connections
```

### Columns:
| Region | Peering ID | Status | Requester VPC | Requester CIDR | Accepter VPC | Accepter CIDR | Tags |

---

## Category 39: Load Balancers ⚡

**Sheet**: `LoadBalancers`

### API Calls:
```
elbv2:DescribeLoadBalancers → ALB/NLB/GWLB
elb:DescribeLoadBalancers → Classic LBs
```

### Columns:
| Region | Name | Type (ALB/NLB/CLB/GWLB) | Scheme | DNS Name | VPC ID | State | AZs | Created | Tags |

---

## Category 40: CloudFront Distributions ⚡ (Global)

**Sheet**: `CloudFront`

### API Calls:
```
cloudfront:ListDistributions → all distributions
```

### Columns:
| Distribution ID | Domain Name | Status | Enabled | Origins | Price Class | HTTP Version | Created | Tags |

---

## Category 41: Route 53 Hosted Zones ⚡ (Global)

**Sheet**: `Route53`

### API Calls:
```
route53:ListHostedZones → all zones
For each zone:
  route53:GetHostedZone (Id=<zoneId>) → details + record count
```

### Columns:
| Zone ID | Zone Name | Type (Public/Private) | Record Count | Comment | Tags |

---

## Category 42: API Gateway

**Sheet**: `APIGateway`

### API Calls:
```
apigateway:GetRestApis → REST APIs
apigatewayv2:GetApis → HTTP & WebSocket APIs
```

### Columns:
| Region | API Name | API ID | Type (REST/HTTP/WebSocket) | Endpoint Type | Created | Tags |

---

## Category 43: Global Accelerator (Global)

**Sheet**: `GlobalAccelerator`

### API Calls:
```
globalaccelerator:ListAccelerators → all accelerators
For each accelerator:
  globalaccelerator:ListListeners (AcceleratorArn=<arn>) → listeners
```

### Columns:
| Name | ARN | Status | DNS Name | IP Addresses | Enabled | Listeners | Created | Tags |

---

## Category 44: Transit Gateways

**Sheet**: `TransitGateway`

### API Calls:
```
ec2:DescribeTransitGateways → all TGWs
ec2:DescribeTransitGatewayAttachments → attachments
```

### Columns:
| Region | TGW ID | Name | State | ASN | Attachments Count | Auto Accept | Created | Tags |

---

## Category 45: Direct Connect

**Sheet**: `DirectConnect`

### API Calls:
```
directconnect:DescribeConnections → all connections
directconnect:DescribeVirtualInterfaces → virtual interfaces
```

### Columns:
| Region | Resource Type | Connection/VIF ID | Name | State | Bandwidth | Location | VLAN | Tags |

---

## Category 46: VPN Connections

**Sheet**: `VPN`

### API Calls:
```
ec2:DescribeVpnConnections → all VPN connections
ec2:DescribeCustomerGateways → customer gateways
```

### Columns:
| Region | Resource Type | ID | Name | State | Type | Customer Gateway | VPN Gateway | Tags |

---

## Category 47: PrivateLink Endpoints

**Sheet**: `PrivateLink`

### API Calls:
```
ec2:DescribeVpcEndpointServices (filter: Owner=self) → owned endpoint services
ec2:DescribeVpcEndpointServiceConfigurations → service configurations
```

### Columns:
| Region | Service ID | Service Name | State | Acceptance Required | LB ARNs | AZs | Tags |

---

## Category 48: Network Firewall

**Sheet**: `NetworkFirewall`

### API Calls:
```
network-firewall:ListFirewalls → firewall list
For each firewall:
  network-firewall:DescribeFirewall (FirewallArn=<arn>) → details
network-firewall:ListFirewallPolicies → policies
network-firewall:ListRuleGroups → rule groups
```

### Columns:
| Region | Resource Type | Name | ARN | Status | VPC ID | Subnet Mappings | Policy | Tags |

---

## Category 49: Elastic IPs

**Sheet**: `ElasticIP`

### API Calls:
```
ec2:DescribeAddresses → all Elastic IPs
```

### Columns:
| Region | Allocation ID | Public IP | Association ID | Instance ID | Network Interface | Domain | Tags |

---

# Group 6: Security (Per Region/Global)

---

## Category 50: KMS Keys

**Sheet**: `KMS`

### API Calls:
```
kms:ListKeys → key IDs
For each key:
  kms:DescribeKey (KeyId=<id>) → key metadata
  kms:ListAliases (KeyId=<id>) → aliases
```

### Columns:
| Region | Key ID | Alias | Status | Key Manager (AWS/Customer) | Key Spec | Created | Tags |

---

## Category 51: Secrets Manager

**Sheet**: `SecretsManager`

### API Calls:
```
secretsmanager:ListSecrets → secret metadata (NEVER retrieve secret values)
```

### Columns:
| Region | Secret Name | ARN | Last Accessed | Last Changed | Rotation Enabled | Rotation Days | Tags |

---

## Category 52: WAF Web ACLs

**Sheet**: `WAF`

### API Calls:
```
wafv2:ListWebACLs (Scope=REGIONAL) → regional WAFs
wafv2:ListWebACLs (Scope=CLOUDFRONT) → global WAFs (us-east-1 only)
```

### Columns:
| Region/Scope | Name | ID | ARN | Rules Count | Default Action | Tags |

---

## Category 53: Shield Protections (Global)

**Sheet**: `Shield`

### API Calls:
```
shield:ListProtections → all protected resources
shield:DescribeSubscription → subscription status
```

### Columns:
| Resource ARN | Resource Type | Protection ID | Name | Health Check | Tags |

---

## Category 54: GuardDuty Detectors

**Sheet**: `GuardDuty`

### API Calls:
```
guardduty:ListDetectors → detector IDs
For each detector:
  guardduty:GetDetector (DetectorId=<id>) → details
  guardduty:GetFindingsStatistics (DetectorId=<id>, FindingCriteria={}) → finding counts
```

### Columns:
| Region | Detector ID | Status | Finding Publishing Frequency | Data Sources | Findings Count | Updated | Tags |

---

## Category 55: Security Hub

**Sheet**: `SecurityHub`

### API Calls:
```
securityhub:DescribeHub → hub status
securityhub:GetEnabledStandards → enabled standards
securityhub:GetFindings (MaxResults=1, SortCriteria=[{Field=UpdatedAt,SortOrder=desc}]) → latest finding sample (count only)
```

### Columns:
| Region | Hub ARN | Status | Standards Enabled | Auto-Enable Controls | Subscribed At | Tags |

---

## Category 56: Inspector

**Sheet**: `Inspector`

### API Calls:
```
inspector2:BatchGetAccountStatus → inspector status
inspector2:ListCoverageStatistics → coverage info
```

### Columns:
| Region | Status | EC2 Scanning | ECR Scanning | Lambda Scanning | Active Findings | Tags |

---

## Category 57: Macie

**Sheet**: `Macie`

### API Calls:
```
macie2:GetMacieSession → session status
macie2:GetBucketStatistics → bucket coverage
```

### Columns:
| Region | Status | Buckets Monitored | Classifiable Objects | Total Findings | Updated |

---

## Category 58: ACM Certificates

**Sheet**: `ACM`

### API Calls:
```
acm:ListCertificates → all certificates
For each certificate:
  acm:DescribeCertificate (CertificateArn=<arn>) → details
```

### Columns:
| Region | Domain Name | ARN | Status | Type (Imported/Amazon Issued) | Key Algorithm | Not After | In Use By | Tags |

---

## Category 59: Firewall Manager Policies

**Sheet**: `Firewall-Manager`

### API Calls:
```
fms:ListPolicies → all FMS policies
```

### Columns:
| Region | Policy Name | Policy ID | Type | Resource Type | Remediation Enabled | Tags |

---

## Category 60: IAM Access Analyzer

**Sheet**: `IAM-AccessAnalyzer`

### API Calls:
```
accessanalyzer:ListAnalyzers → all analyzers
For each analyzer:
  accessanalyzer:ListFindings (analyzerArn=<arn>, maxResults=10) → sample findings count
```

### Columns:
| Region | Analyzer Name | ARN | Type | Status | Active Findings | Created | Tags |

---

# Group 7: Application Integration (Per Region)

---

## Category 61: SQS Queues

**Sheet**: `SQS`

### API Calls:
```
sqs:ListQueues → queue URLs
For each queue:
  sqs:GetQueueAttributes (AttributeNames=All) → details
```

### Columns:
| Region | Queue Name | Queue URL | Type (Standard/FIFO) | Messages Available | Messages In Flight | Created | Tags |

---

## Category 62: SNS Topics

**Sheet**: `SNS`

### API Calls:
```
sns:ListTopics → topic ARNs
For each topic:
  sns:GetTopicAttributes (TopicArn=<arn>) → details
```

### Columns:
| Region | Topic Name | Topic ARN | Subscriptions Count | Encryption | Display Name | Tags |

---

## Category 63: EventBridge Rules & Buses

**Sheet**: `EventBridge`

### API Calls:
```
events:ListEventBuses → event buses
For each bus:
  events:ListRules (EventBusName=<name>) → rules
```

### Columns:
| Region | Bus Name | Rule Name | State | Schedule/Event Pattern | Targets Count | Tags |

---

## Category 64: Step Functions

**Sheet**: `StepFunctions`

### API Calls:
```
sfn:ListStateMachines → state machine list
```

### Columns:
| Region | Name | ARN | Type (STANDARD/EXPRESS) | Status | Created | Tags |

---

## Category 65: AppSync APIs

**Sheet**: `AppSync`

### API Calls:
```
appsync:ListGraphqlApis → all GraphQL APIs
```

### Columns:
| Region | API Name | API ID | Authentication Type | URIs | Status | Created | Tags |

---

## Category 66: Amazon MQ Brokers

**Sheet**: `MQ`

### API Calls:
```
mq:ListBrokers → broker summaries
For each broker:
  mq:DescribeBroker (BrokerId=<id>) → details
```

### Columns:
| Region | Broker Name | Broker ID | Engine | Engine Version | Instance Type | Deployment Mode | State | Created | Tags |

---

## Category 67: SES Identities

**Sheet**: `SES`

### API Calls:
```
sesv2:ListEmailIdentities → all verified identities
sesv2:ListConfigurationSets → config sets
```

### Columns:
| Region | Resource Type | Identity/Config Name | Type (Email/Domain) | Status | DKIM | Tags |

---

# Group 8: Analytics (Per Region)

---

## Category 68: Kinesis Streams

**Sheet**: `Kinesis`

### API Calls:
```
kinesis:ListStreams → stream names
For each stream:
  kinesis:DescribeStreamSummary (StreamName=<name>) → details
```

### Columns:
| Region | Stream Name | Status | Shard Count | Retention (Hours) | Encryption | Mode | Created | Tags |

---

## Category 69: MSK Clusters

**Sheet**: `MSK`

### API Calls:
```
kafka:ListClustersV2 → cluster summaries
```

### Columns:
| Region | Cluster Name | Cluster ARN | Type (Provisioned/Serverless) | State | Kafka Version | Broker Nodes | Created | Tags |

---

## Category 70: Glue Jobs & Crawlers

**Sheet**: `Glue`

### API Calls:
```
glue:GetJobs → job list
glue:GetCrawlers → crawler list
glue:GetDatabases → database list
```

### Columns:
| Region | Resource Type (Job/Crawler/Database) | Name | Status/State | Last Run | Created | Tags |

---

## Category 71: Athena Workgroups

**Sheet**: `Athena`

### API Calls:
```
athena:ListWorkGroups → workgroup list
```

### Columns:
| Region | Workgroup Name | State | Engine Version | Output Location | Enforce Config | Tags |

---

## Category 72: EMR Clusters

**Sheet**: `EMR`

### API Calls:
```
emr:ListClusters (ClusterStates=STARTING,BOOTSTRAPPING,RUNNING,WAITING,TERMINATING) → active clusters
```

### Columns:
| Region | Cluster ID | Name | Status | Instance Hours | Created | Tags |

---

## Category 73: QuickSight Datasets

**Sheet**: `QuickSight`

### API Calls:
```
quicksight:ListDataSets (AwsAccountId=<accountId>) → datasets
quicksight:ListDashboards (AwsAccountId=<accountId>) → dashboards
```

### Columns:
| Region | Resource Type | Name | ID | Status | Created | Last Updated | Tags |

---

## Category 74: Data Pipeline

**Sheet**: `DataPipeline`

### API Calls:
```
datapipeline:ListPipelines → pipeline list
For each pipeline:
  datapipeline:DescribePipelines (pipelineIds=[<id>]) → details
```

### Columns:
| Region | Pipeline Name | Pipeline ID | State | Created | Tags |

---

## Category 75: Lake Formation Resources

**Sheet**: `LakeFormation`

### API Calls:
```
lakeformation:ListResources → registered data lake locations
lakeformation:GetDataLakeSettings → settings
```

### Columns:
| Region | Resource ARN | Role ARN | Last Modified | Tags |

---

## Category 76: OpenSearch Serverless Collections

**Sheet**: `OpenSearchServerless`

### API Calls:
```
opensearchserverless:ListCollections → collection list
```

### Columns:
| Region | Collection Name | Collection ID | Type | Status | ARN | Created | Tags |

---

# Group 9: Machine Learning (Per Region)

---

## Category 77: SageMaker Endpoints & Notebooks

**Sheet**: `SageMaker`

### API Calls:
```
sagemaker:ListEndpoints → endpoint list
sagemaker:ListNotebookInstances → notebook instances
sagemaker:ListTrainingJobs (StatusEquals=InProgress) → active training jobs
```

### Columns:
| Region | Resource Type (Endpoint/Notebook/TrainingJob) | Name | Status | Instance Type | Created | Tags |

---

## Category 78: Bedrock Models & Agents

**Sheet**: `Bedrock`

### API Calls:
```
bedrock:ListFoundationModels → available models
bedrock:ListCustomModels → custom models
bedrock:ListAgents → agents
bedrock:ListKnowledgeBases → knowledge bases
```

### Columns:
| Region | Resource Type | Name/Model ID | Status | Provider | Created | Tags |

---

## Category 79: Comprehend Endpoints

**Sheet**: `Comprehend`

### API Calls:
```
comprehend:ListEndpoints → all endpoints
comprehend:ListDocumentClassifiers → classifiers
```

### Columns:
| Region | Resource Type | Name | ARN | Status | Model ARN | Created | Tags |

---

## Category 80: Rekognition Collections

**Sheet**: `Rekognition`

### API Calls:
```
rekognition:ListCollections → collection IDs
For each collection:
  rekognition:DescribeCollection (CollectionId=<id>) → details
```

### Columns:
| Region | Collection ID | ARN | Face Count | Created | Tags |

---

## Category 81: Textract Adapters

**Sheet**: `Textract`

### API Calls:
```
textract:ListAdapters → adapter list
```

### Columns:
| Region | Adapter Name | Adapter ID | Status | Feature Types | Created | Tags |

---

## Category 82: Transcribe Jobs

**Sheet**: `Transcribe`

### API Calls:
```
transcribe:ListTranscriptionJobs (Status=COMPLETED) → completed jobs (last 100)
transcribe:ListVocabularies → custom vocabularies
transcribe:ListLanguageModels → custom language models
```

### Columns:
| Region | Resource Type | Name | Status | Language | Created | Tags |

---

## Category 83: Polly Lexicons

**Sheet**: `Polly`

### API Calls:
```
polly:ListLexicons → all lexicons
```

### Columns:
| Region | Lexicon Name | Language | Size (bytes) | Alphabet | Last Modified |

---

## Category 84: Forecast Datasets

**Sheet**: `Forecast`

### API Calls:
```
forecast:ListDatasets → all datasets
forecast:ListPredictors → all predictors
```

### Columns:
| Region | Resource Type | Name | ARN | Status | Domain | Created | Tags |

---

# Group 10: Management & Monitoring (Per Region/Global)

---

## Category 85: CloudWatch Alarms

**Sheet**: `CloudWatch`

### API Calls:
```
cloudwatch:DescribeAlarms → all alarms
```

### Columns:
| Region | Alarm Name | Namespace | Metric | State | Actions Enabled | Threshold | Period | Tags |

---

## Category 86: CloudFormation Stacks

**Sheet**: `CloudFormation`

### API Calls:
```
cloudformation:ListStacks (StackStatusFilter=CREATE_COMPLETE,UPDATE_COMPLETE,ROLLBACK_COMPLETE,UPDATE_ROLLBACK_COMPLETE) → active stacks
```

### Columns:
| Region | Stack Name | Stack ID | Status | Created | Last Updated | Drift Status | Template Description | Tags |

---

## Category 87: CloudTrail Trails

**Sheet**: `CloudTrail`

### API Calls:
```
cloudtrail:DescribeTrails → all trails
For each trail:
  cloudtrail:GetTrailStatus (Name=<trailARN>) → status
```

### Columns:
| Region | Trail Name | ARN | Multi-Region | Is Organization | S3 Bucket | Log Validation | Logging | Latest Delivery | Tags |

---

## Category 88: AWS Config Rules

**Sheet**: `Config`

### API Calls:
```
config:DescribeConfigRules → all config rules
config:DescribeComplianceByConfigRule → compliance status
config:DescribeConfigurationRecorders → recorders
```

### Columns:
| Region | Resource Type | Rule/Recorder Name | State | Compliance Status | Source | Tags |

---

## Category 89: Systems Manager

**Sheet**: `SystemsManager`

### API Calls:
```
ssm:DescribeInstanceInformation → managed instances
ssm:ListDocuments (Filters=[{Key=Owner,Values=[Self]}]) → custom documents
ssm:ListAssociations → associations
ssm:DescribeMaintenanceWindows → maintenance windows
ssm:GetParametersByPath (Path=/, Recursive=false, MaxResults=50) → parameter count (top level only)
```

### Columns:
| Region | Resource Type | Name/Instance ID | Status | Platform | Agent Version | Last Ping | Tags |

---

## Category 90: Trusted Advisor (Global)

**Sheet**: `TrustedAdvisor`

### API Calls:
```
support:DescribeTrustedAdvisorChecks (language=en) → all checks
support:DescribeTrustedAdvisorCheckSummaries (checkIds=<all_ids>) → summaries
```

### Columns:
| Category | Check Name | Status | Resources Flagged | Resources Warning | Resources Error | Timestamp |

---

## Category 91: Health Dashboard (Global)

**Sheet**: `HealthDashboard`

### API Calls:
```
health:DescribeEvents (filter={eventStatusCodes=[open,upcoming]}) → active events
health:DescribeAffectedEntities → affected resources (for open events)
```

### Columns:
| Event Type | Service | Status | Region | Start Time | End Time | Description |

---

## Category 92: Service Catalog

**Sheet**: `ServiceCatalog`

### API Calls:
```
servicecatalog:ListPortfolios → portfolios
servicecatalog:SearchProducts → products
```

### Columns:
| Region | Resource Type | Name | ID | Status | Provider | Created | Tags |

---

# Group 11: Developer Tools (Per Region)

---

## Category 93: CodeCommit Repositories

**Sheet**: `CodeCommit`

### API Calls:
```
codecommit:ListRepositories → repository names
For each repo:
  codecommit:GetRepository (repositoryName=<name>) → details
```

### Columns:
| Region | Repository Name | ARN | Clone URL (HTTPS) | Default Branch | Last Modified | Tags |

---

## Category 94: CodeBuild Projects

**Sheet**: `CodeBuild`

### API Calls:
```
codebuild:ListProjects → project names
codebuild:BatchGetProjects (names=<project_names>) → project details
```

### Columns:
| Region | Project Name | ARN | Source Type | Environment Type | Compute Type | Last Build Status | Created | Tags |

---

## Category 95: CodeDeploy Applications

**Sheet**: `CodeDeploy`

### API Calls:
```
codedeploy:ListApplications → app names
codedeploy:BatchGetApplications (applicationNames=<names>) → details
codedeploy:ListDeploymentGroups (applicationName=<name>) → deployment groups per app
```

### Columns:
| Region | Application Name | Compute Platform | Deployment Groups | Created | Tags |

---

## Category 96: CodePipeline Pipelines

**Sheet**: `CodePipeline`

### API Calls:
```
codepipeline:ListPipelines → pipeline summaries
```

### Columns:
| Region | Pipeline Name | ARN | Version | Stage Count | Created | Updated | Tags |

---

## Category 97: CodeArtifact Repositories

**Sheet**: `CodeArtifact`

### API Calls:
```
codeartifact:ListDomains → domains
codeartifact:ListRepositories → repositories
```

### Columns:
| Region | Resource Type | Domain/Repo Name | ARN | Status | Created | Tags |

---

## Category 98: Cloud9 Environments

**Sheet**: `Cloud9`

### API Calls:
```
cloud9:ListEnvironments → environment IDs
cloud9:DescribeEnvironments (environmentIds=<ids>) → details
```

### Columns:
| Region | Environment Name | ID | Type | Connection Type | Instance Type | Status | Created | Tags |

---

## Category 99: ECR Repositories

**Sheet**: `ECR`

### API Calls:
```
ecr:DescribeRepositories → all repos
For each repo:
  ecr:ListImages (repositoryName=<name>, maxResults=1) → image count check
```

### Columns:
| Region | Repository Name | ARN | URI | Image Tag Mutability | Scan on Push | Encryption | Created | Tags |

---

# Group 12: Migration & Transfer (Per Region)

---

## Category 100: DMS Replication Instances

**Sheet**: `DMS`

### API Calls:
```
dms:DescribeReplicationInstances → replication instances
dms:DescribeReplicationTasks → replication tasks
dms:DescribeEndpoints → endpoints
```

### Columns:
| Region | Resource Type | Identifier | Class/Type | Status | Engine | Endpoint/VPC | Created | Tags |

---

## Category 101: Transfer Family Servers

**Sheet**: `TransferFamily`

### API Calls:
```
transfer:ListServers → all servers
For each server:
  transfer:DescribeServer (ServerId=<id>) → details
```

### Columns:
| Region | Server ID | Endpoint Type | Protocol | Domain | State | Users | Identity Provider | Tags |

---

## Category 102: DataSync Tasks

**Sheet**: `DataSync`

### API Calls:
```
datasync:ListTasks → all tasks
datasync:ListLocations → all locations
```

### Columns:
| Region | Resource Type | Name | ARN | Status | Source Location | Destination Location | Tags |

---

## Category 103: Migration Hub

**Sheet**: `MigrationHub`

### API Calls:
```
mgh:ListMigrationTasks → migration tasks
mgh:ListApplicationStates → application states
```

### Columns:
| Region | Resource Type | Name | Status | Progress | Update Time | Tags |

---

## Category 104: Snow Family Devices

**Sheet**: `Snow`

### API Calls:
```
snowball:ListJobs → all Snow jobs
snowball:ListClusters → Snow clusters
```

### Columns:
| Region | Resource Type | Job/Cluster ID | Type | State | Created | Description | Tags |

---

# Group 13: Cost Management (Global)

---

## Category 105: Cost Summary (Optional)

**Sheet**: `CostSummary`

Only if user opted in and has Cost Explorer access.

### API Calls:
```
ce:GetCostAndUsage (TimePeriod=last 30 days, Granularity=MONTHLY, Metrics=UnblendedCost, GroupBy=SERVICE) → cost by service
```

### Columns:
| Service | Cost (USD) | Unit | Time Period |

---

## Category 106: Budgets

**Sheet**: `Budgets`

### API Calls:
```
budgets:DescribeBudgets (AccountId=<accountId>) → all budgets
```

### Columns:
| Budget Name | Type | Limit | Actual Spend | Forecasted Spend | Time Period | Alerts | Tags |

---

## Category 107: Cost Anomaly Detection

**Sheet**: `CostAnomalyDetection`

### API Calls:
```
ce:GetAnomalyMonitors → monitors
ce:GetAnomalySubscriptions → subscriptions
ce:GetAnomalies (DateInterval={StartDate=<30 days ago>, EndDate=<today>}) → recent anomalies
```

### Columns:
| Resource Type | Name/ID | Status | Type | Anomaly Score | Impact (USD) | Start Date | End Date |

---

## Category 108: Savings Plans

**Sheet**: `SavingsPlans`

### API Calls:
```
savingsplans:DescribeSavingsPlans → all savings plans
```

### Columns:
| Savings Plan ID | Type | State | Commitment (USD/hr) | Start | End | Utilization | Tags |

---

## Category 109: Reserved Instances

**Sheet**: `ReservedInstances`

### API Calls:
```
ec2:DescribeReservedInstances → EC2 RIs
rds:DescribeReservedDBInstances → RDS RIs
elasticache:DescribeReservedCacheNodes → ElastiCache RIs
redshift:DescribeReservedNodes → Redshift RIs
opensearch:DescribeReservedInstances → OpenSearch RIs
```

### Columns:
| Region | Service | Instance Type | Count | State | Offering Type | Start | End | Recurring Charge | Tags |

---

# Scan Completion

After all categories are scanned:
1. Aggregate all collected data into `inventory-reports/inventory-data.json`.
2. Run `python scripts/generate_excel.py` to produce the workbook.
3. Clean up the temporary JSON file.
4. Report the final file path and summary to the user.

---

# Quick Scan Categories ⚡

When `scanMode` is `quick`, only the following categories are executed:

| # | Category | Sheet |
|---|----------|-------|
| 4 | EC2 Instances | EC2-Instances |
| 5 | EC2 Security Groups | EC2-SecurityGroups |
| 10 | Lambda Functions | Lambda |
| 11 | ECS Clusters & Services | ECS |
| 12 | EKS Clusters | EKS |
| 18 | S3 Buckets | S3 |
| 19 | EBS Volumes | EBS |
| 20 | EFS File Systems | EFS |
| 36 | VPC & Networking | VPC |
| 39 | Load Balancers | LoadBalancers |
| 40 | CloudFront Distributions | CloudFront |
| 41 | Route 53 Hosted Zones | Route53 |

This provides a fast infrastructure overview covering compute, storage, and networking fundamentals.
