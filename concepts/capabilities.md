# Capabilities

## What is a Capability?

A **capability** is a functional infrastructure abstraction that describes *what* you want to accomplish, not *how* to implement it.

Traditional thinking (implementation-focused):
> "I need an ECS cluster with Fargate, an Application Load Balancer, a VPC with 2 public subnets..."

Capability thinking (outcome-focused):
> "I need to run a web application"

## Why Capabilities?

### Problem with Resource-Based Thinking

```
User wants: "Deploy my web app"

AWS approach:
- Learn about: VPC, Subnets, Internet Gateway, Route Tables, Security Groups,
  ECS Clusters, Task Definitions, Services, Target Groups, Load Balancers,
  ECR repositories, IAM roles, CloudWatch logs...
- Configure 20+ resources
- Debug networking, permissions, health checks...

This is overwhelming and error-prone.
```

### Capability-Based Approach

```
User specifies:
- Capability: web_application
- Configuration:
  * container_image: myapp:latest
  * cpu: 2 vCPU
  * memory: 4GB
  * scaling: 1-10 instances

Infrar handles:
- Select best implementation (AWS ECS, GCP Cloud Run, or Azure Container Apps)
- Provision all required resources
- Configure networking, security, monitoring
- Deploy application
```

## Capability Hierarchy

```
Category
├── Capability
│   ├── Implementation (Provider A)
│   ├── Implementation (Provider B)
│   └── Implementation (Provider C)
```

Example:

```
Compute
├── web_application
│   ├── aws_ecs_fargate (ECS + ALB + VPC)
│   ├── gcp_cloud_run (Cloud Run + Load Balancing)
│   └── azure_container_apps (Container Apps + Gateway)
│
├── api_service
│   ├── aws_lambda_api_gateway (Lambda + API Gateway)
│   ├── gcp_cloud_functions (Cloud Functions + API Gateway)
│   └── azure_functions (Azure Functions + API Management)
│
└── background_worker
    ├── aws_ecs_worker (ECS without ALB)
    ├── gcp_cloud_run_jobs (Cloud Run Jobs)
    └── azure_container_apps_jobs (Container Apps Jobs)

Data
├── data_warehouse
│   ├── aws_athena_glue (Athena + Glue + S3)
│   ├── gcp_bigquery (BigQuery + Cloud Storage)
│   └── azure_synapse (Synapse + Blob Storage)
│
└── relational_database
    ├── aws_rds_postgres (RDS PostgreSQL)
    ├── gcp_cloud_sql_postgres (Cloud SQL PostgreSQL)
    └── azure_database_postgres (Azure Database for PostgreSQL)

Storage
└── object_storage
    ├── aws_s3 (S3)
    ├── gcp_cloud_storage (Cloud Storage)
    └── azure_blob_storage (Blob Storage)
```

## Capability Properties

Each capability defines:

### 1. What It Provides

Functional features regardless of provider:

```yaml
provides:
  - http_server
  - https_termination
  - auto_scaling
  - health_checks
  - zero_downtime_deployment
```

### 2. How It's Measured

Metrics for cost calculation and sizing:

```yaml
metrics:
  - cpu_units: "Number of vCPUs"
  - memory_mb: "Memory in megabytes"
  - request_count_month: "HTTP requests per month"
  - data_transfer_gb_month: "Egress data in GB"
```

### 3. Configuration

User-facing options:

```yaml
configuration:
  - container_image: "Docker image"
  - container_port: "Port to expose (default: 8080)"
  - environment_variables: "Environment vars"
  - min_instances: "Minimum replicas (default: 1)"
  - max_instances: "Maximum replicas (default: 10)"
```

### 4. Implementations

Provider-specific ways to achieve the capability:

```yaml
implementations:
  aws_ecs_fargate:
    provider: aws
    services: [ecs, alb, vpc]
    cost_model: ...
    capabilities_coverage: ...

  gcp_cloud_run:
    provider: gcp
    services: [cloud_run]
    cost_model: ...
    capabilities_coverage: ...
```

## Capability Examples

### Example 1: Web Application

```yaml
name: web_application
provides:
  - http_server
  - auto_scaling
  - load_balancing

configuration:
  container_image: "myorg/webapp:v1"
  cpu: 2
  memory: 4096
  min_instances: 2
  max_instances: 10

# Infrar selects implementation
selected_implementation: gcp_cloud_run  # Based on cost/performance

result:
  - GCP Cloud Run service created
  - Load balancer configured
  - Auto-scaling enabled
  - HTTPS endpoint: https://webapp-abc123.run.app
```

### Example 2: Data Intelligence

```yaml
name: data_intelligence
provides:
  - sql_query_engine
  - etl_orchestration
  - data_catalog

configuration:
  data_format: parquet
  data_size_tb: 5
  query_frequency: daily
  concurrent_queries: 10

# Infrar selects implementation
selected_implementation: azure_synapse  # Best cost for this workload

result:
  - Azure Synapse workspace created
  - Data Factory pipelines configured
  - Blob Storage connected
  - Query endpoint: synapse-workspace.sql.azuresynapse.net
```

## Capability Selection

### User Perspective

```
User: "I want to run a web application"

Infrar asks:
1. What's your expected traffic? (10k, 100k, 1M requests/day?)
2. What resources do you need? (CPU, memory)
3. Any region preferences?
4. Budget constraints?

Infrar calculates:
- AWS ECS Fargate: $127/month
- GCP Cloud Run: $98/month ✅ Recommended
- Azure Container Apps: $106/month

User selects: GCP Cloud Run
```

### Automated Selection (Discovery)

```
Infrar scans AWS account:
- Found: ECS Cluster + ECS Service + ALB
- Mapped to: web_application capability
- Implementation: aws_ecs_fargate
- Current cost: $127/month

Infrar suggests alternatives:
- Move to GCP Cloud Run: $98/month (23% savings)
- Move to Azure Container Apps: $106/month (16% savings)
```

## Capability Composition

Combine multiple capabilities for complete applications:

```yaml
project: ecommerce_platform

capabilities:
  - name: web_frontend
    capability: web_application
    config:
      container_image: myorg/frontend:latest
      cpu: 1
      memory: 2048

  - name: api_backend
    capability: api_service
    config:
      container_image: myorg/backend:latest
      cpu: 2
      memory: 4096

  - name: database
    capability: relational_database
    config:
      engine: postgres
      version: "15"
      storage_gb: 100

  - name: file_storage
    capability: object_storage
    config:
      storage_class: standard

# Infrar can deploy each capability on optimal provider
recommendations:
  web_frontend: gcp_cloud_run ($45/mo)
  api_backend: aws_lambda ($78/mo)
  database: azure_postgres ($95/mo)
  file_storage: aws_s3 ($23/mo)

total_cost: $241/month
vs_single_provider: $310/month (AWS all)
savings: 22%
```

## Capability vs. Service vs. Resource

**Resource** (lowest level):
- Cloud-specific implementation detail
- Example: `aws_ecs_service`, `aws_alb`, `aws_vpc`

**Service** (mid level):
- Cloud-specific product
- Example: "AWS ECS", "GCP Cloud Run", "Azure Container Apps"

**Capability** (highest level):
- Cloud-agnostic functional requirement
- Example: "web_application", "data_warehouse"

```
Capability: web_application
    ↓ Maps to Service
Service: AWS ECS (or GCP Cloud Run, or Azure Container Apps)
    ↓ Provisions Resources
Resources: ECS Cluster, Task Definition, Service, ALB, VPC, etc.
```

## Benefits of Capability-Based Abstraction

### 1. Simplified User Experience

**Before (resource-based)**:
- User needs to understand 20+ AWS resources
- Configure each resource individually
- Debug interactions between resources

**After (capability-based)**:
- User describes what they want ("web application")
- Infrar handles all resource provisioning
- Best practices applied automatically

### 2. Provider Flexibility

**Before**:
- Locked into AWS ECS
- Hard to compare costs with GCP/Azure
- Migration requires rewriting everything

**After**:
- Select "web_application" capability
- Compare costs across all providers
- Switch providers by regenerating config

### 3. Cost Optimization

**Before**:
- Don't know if another provider is cheaper
- Manual cost comparison is tedious
- Can't optimize individual services

**After**:
- Automatic cost comparison
- Optimize each capability independently
- Data-driven migration decisions

### 4. Evolution Over Time

**Before**:
- Infrastructure becomes outdated
- Hard to adopt new services
- Refactoring is risky

**After**:
- Capabilities evolve (new implementations added)
- Automatically benefit from new providers/services
- Smooth migration path

## Capability Maturity Levels

### Level 1: Basic (MVP)

```yaml
capability: web_application
implementations:
  - aws_ecs_fargate
  - gcp_cloud_run
  - azure_container_apps

features:
  - Deploy container
  - Auto-scaling
  - Load balancing
```

### Level 2: Production

```yaml
capability: web_application
implementations:
  - (all from Level 1)
  - kubernetes (EKS/GKE/AKS)
  - digitalocean_app_platform

features:
  - (all from Level 1)
  - Custom domains
  - SSL certificates
  - CDN integration
  - WAF protection
```

### Level 3: Enterprise

```yaml
capability: web_application
implementations:
  - (all from Level 2)
  - on_premises_kubernetes
  - multi_region

features:
  - (all from Level 2)
  - Multi-region deployment
  - Advanced routing
  - Chaos engineering
  - Compliance certifications
```

## Limitations

### Not All Services Map Cleanly

Some provider-specific services don't have equivalents:
- AWS Lambda@Edge
- GCP Cloud Armor
- Azure Cosmos DB

**Solution**: Offer as provider-specific features

```python
from infrar.aws.edge import lambda_edge  # AWS-only

# User is aware this locks them to AWS
lambda_edge.deploy(...)
```

### Different Operational Models

Cloud Run scales to zero, ECS doesn't:

```yaml
gcp_cloud_run:
  min_instances: 0  # Scale to zero ✅

aws_ecs_fargate:
  min_instances: 1  # Must run at least 1 ❌
```

**Solution**: Capability defines common denominator, provider-specific config allows optimization

## Extending Capabilities

To add a new capability:

1. Define functional requirements
2. Identify metrics for cost calculation
3. Map to provider services
4. Create implementations
5. Define cost models
6. Test with real deployments

Example: Adding `cdn` capability

```yaml
name: cdn
category: networking
provides:
  - content_caching
  - global_distribution
  - ddos_protection

implementations:
  aws_cloudfront: ...
  gcp_cloud_cdn: ...
  azure_cdn: ...
  cloudflare: ...
```

## Conclusion

Capabilities are the foundation of Infrar's abstraction:
- **Functional**: Focus on outcomes, not implementation
- **Provider-agnostic**: Same capability, multiple implementations
- **Cost-aware**: Compare prices across providers
- **Evolvable**: Add new implementations over time

Next: [Plugin System](plugins.md) - How capabilities are implemented as plugins
