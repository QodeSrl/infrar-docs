# Architecture Overview

## System Design

Infrar is designed as a multi-layered platform that sits between user code and cloud providers, enabling infrastructure intelligence and seamless provider switching.

**Note**: For details about the codebase organization and individual repositories, see [Repository Structure](./repository-structure.md).

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      User Layer                              │
│                                                              │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │  GitHub Repo     │         │  Local Dev Env   │         │
│  │  (infrar SDK)    │         │  (infrar SDK)    │         │
│  └────────┬─────────┘         └──────────────────┘         │
└───────────┼──────────────────────────────────────────────────┘
            │ Webhook on push
            ↓
┌──────────────────────────────────────────────────────────────┐
│                   Infrar Platform                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │               Web UI / API Gateway                   │   │
│  │  - Authentication & Authorization                    │   │
│  │  - Project Management                                │   │
│  │  - Deployment Dashboard                              │   │
│  └──────────────────┬───────────────────────────────────┘   │
│                     │                                        │
│  ┌──────────────────▼───────────────────────────────────┐   │
│  │            Core Backend Services                     │   │
│  │                                                      │   │
│  │  ┌────────────────────┐  ┌─────────────────────┐   │   │
│  │  │ Discovery Engine   │  │ Cost Intelligence   │   │   │
│  │  │ - Scan cloud       │  │ - Pricing database  │   │   │
│  │  │ - Map capabilities │  │ - Compare providers │   │   │
│  │  └────────────────────┘  └─────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌────────────────────┐  ┌─────────────────────┐   │   │
│  │  │ Transformation     │  │ Capability Catalog  │   │   │
│  │  │ Engine             │  │ - Definitions       │   │   │
│  │  │ - AST parsing      │  │ - Implementations   │   │   │
│  │  │ - Code generation  │  │ - Cost models       │   │   │
│  │  └────────────────────┘  └─────────────────────┘   │   │
│  │                                                      │   │
│  │  ┌────────────────────┐  ┌─────────────────────┐   │   │
│  │  │ Build & Deploy     │  │ Migration           │   │   │
│  │  │ Pipeline           │  │ Orchestrator        │   │   │
│  │  │ - Build Docker     │  │ - Plan migrations   │   │   │
│  │  │ - Push to registry │  │ - Execute cutover   │   │   │
│  │  │ - Apply OpenTofu   │  │ - Track progress    │   │   │
│  │  └────────────────────┘  └─────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Data Layer                              │   │
│  │                                                      │   │
│  │  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ PostgreSQL │  │ Object      │  │ Credential  │  │   │
│  │  │ - Projects │  │ Storage     │  │ Vault       │  │   │
│  │  │ - Deploys  │  │ - Configs   │  │ - Provider  │  │   │
│  │  │ - Users    │  │ - States    │  │   keys      │  │   │
│  │  └────────────┘  └─────────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬──────────────────────────────────┘
                            │ Deploy via credentials
                            ↓
┌──────────────────────────────────────────────────────────────┐
│                  Cloud Provider Layer                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │     AWS      │  │     GCP      │  │    Azure     │      │
│  │              │  │              │  │              │      │
│  │ Customer     │  │ Customer     │  │ Customer     │      │
│  │ Account      │  │ Project      │  │ Subscription │      │
│  │              │  │              │  │              │      │
│  │ Infrastructure│  │ Infrastructure│  │ Infrastructure│    │
│  │ (Native SDK) │  │ (Native SDK) │  │ (Native SDK) │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Web UI / API Gateway

**Purpose**: User-facing interface for managing projects and infrastructure

**Responsibilities**:
- Authentication and authorization
- Project creation and management
- Provider selection and credential setup
- Deployment triggering and monitoring
- Cost visualization and comparison
- Migration planning interface

**Technology**: React frontend, Go backend API

### 2. Discovery Engine

**Purpose**: Analyze existing cloud infrastructure

**Responsibilities**:
- Connect to cloud accounts (read-only for discovery)
- Scan resources (EC2, ECS, Lambda, RDS, etc.)
- Identify resource relationships
- Map resources to capabilities
- Calculate current costs from billing data
- Detect application dependencies

**Inputs**: Cloud provider credentials (read-only)
**Outputs**: Capability map, cost breakdown, resource inventory

### 3. Cost Intelligence Engine

**Purpose**: Provide pricing insights and comparisons

**Responsibilities**:
- Maintain pricing database for all providers
- Calculate total cost of ownership (compute + storage + network + extras)
- Compare equivalent implementations across providers
- Generate cost projections
- Track pricing changes over time
- Calculate migration ROI

**Data Sources**:
- Provider pricing APIs (AWS Price List API, GCP Cloud Billing API, Azure Pricing API)
- Historical billing data
- Usage metrics

### 4. Transformation Engine

**Purpose**: Convert infrar SDK code to provider-specific code

**Responsibilities**:
- Parse user code (AST analysis)
- Identify infrar SDK calls
- Apply transformation rules
- Generate provider-specific code
- Validate generated code
- Handle edge cases and errors

**Inputs**: Source code with infrar SDK
**Outputs**: Transformed code with provider SDK

See: [Transformation Engine Details](transformation-engine.md)

### 5. Capability Catalog

**Purpose**: Define infrastructure capabilities and their implementations

**Responsibilities**:
- Store capability definitions
- Map capabilities to provider services
- Define cost models for each implementation
- Specify feature coverage per implementation
- Document cross-cloud interfaces

**Schema**: YAML-based definitions

See: [Capability Catalog Details](capability-catalog.md)

### 6. Build & Deploy Pipeline

**Purpose**: Build and deploy applications to cloud providers

**Responsibilities**:
- Clone user repository
- Transform code (via Transformation Engine)
- Build Docker images
- Push to provider registries (ECR, Artifact Registry, ACR)
- Generate OpenTofu configurations
- Execute `tofu plan` and `tofu apply`
- Monitor deployment status
- Store deployment artifacts

**Workflow**:
```
GitHub Push → Webhook → Clone Repo → Transform Code →
Build Docker → Push to Registry → Generate OpenTofu →
Apply Infrastructure → Deploy Application → Update Status
```

### 7. Migration Orchestrator

**Purpose**: Plan and execute infrastructure migrations

**Responsibilities**:
- Analyze current infrastructure
- Generate migration plan
- Calculate costs and timeline
- Provision target infrastructure (parallel deployment)
- Orchestrate data migration
- Manage DNS cutover
- Monitor migration health
- Rollback on failure
- Cleanup source infrastructure

**Migration Phases**:
1. **Analysis**: Understand current state
2. **Planning**: Generate migration plan with costs/timeline
3. **Provision**: Deploy target infrastructure
4. **Migrate**: Move data and configuration
5. **Test**: Validate target infrastructure
6. **Cutover**: Switch traffic
7. **Monitor**: Ensure stability
8. **Cleanup**: Decommission source

See: [Migration Details](../concepts/migration.md)

### 8. Data Layer

#### PostgreSQL Database

**Schema**:
```sql
-- Users and authentication
users (id, email, created_at, ...)

-- Projects
projects (id, user_id, name, repo_url, created_at, ...)

-- Deployments
deployments (
    id, project_id, provider, region,
    status, created_at, completed_at, ...
)

-- Deployment configurations
deployment_configs (
    deployment_id, capability_type,
    implementation, config_json, ...
)

-- Provider credentials
provider_credentials (
    id, user_id, provider,
    encrypted_credentials, created_at, ...
)

-- Cost history
cost_snapshots (
    id, deployment_id, cost,
    snapshot_date, breakdown_json, ...
)

-- Migration history
migrations (
    id, project_id, source_provider,
    target_provider, status, started_at, ...
)
```

#### Object Storage

**Structure**:
```
s3://infrar-configs/{project_id}/{deployment_id}/
├── original/
│   └── source_code/       # Original infrar SDK code
├── transformed/
│   └── source_code/       # Provider-specific transformed code
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── logs/
    ├── transform.log
    ├── build.log
    └── deploy.log

s3://infrar-state/{project_id}/{deployment_id}/
└── terraform.tfstate       # OpenTofu state (encrypted)
```

#### Credential Vault

**Purpose**: Securely store provider credentials

**Approach**:
- **MVP**: PostgreSQL with encryption at rest
- **Production**: HashiCorp Vault or cloud KMS
- **Enterprise**: OIDC/Workload Identity (no long-lived keys)

**Security**:
- Encryption at rest (AES-256)
- Encryption in transit (TLS)
- Audit logging for all access
- Credential rotation policies
- Least-privilege IAM policies

See: [Security Details](../technical/security.md)

## Data Flow

### Deployment Flow

```
1. User pushes code to GitHub
   └─> GitHub webhook triggers Infrar

2. Infrar clones repository
   └─> Reads code and Dockerfile

3. Code Analysis
   ├─> Parse AST
   ├─> Identify infrar SDK calls
   └─> Validate code patterns

4. Transformation
   ├─> Apply transformation rules
   ├─> Generate provider-specific code
   └─> Validate generated code

5. Build
   ├─> Build Docker image (with transformed code)
   ├─> Tag image
   └─> Push to provider registry

6. Infrastructure Provisioning
   ├─> Load capability definitions
   ├─> Generate OpenTofu configuration
   ├─> Execute tofu plan
   └─> Execute tofu apply

7. Application Deployment
   ├─> Deploy container to compute service
   ├─> Configure networking
   └─> Set environment variables

8. Validation
   ├─> Health checks
   ├─> Log collection
   └─> Update deployment status

9. Notification
   └─> Notify user (webhook, email, dashboard)
```

### Migration Flow

```
1. User initiates migration
   └─> Selects target provider

2. Current State Analysis
   ├─> Read OpenTofu state
   ├─> Identify all resources
   └─> Calculate current costs

3. Target State Planning
   ├─> Select equivalent capabilities
   ├─> Calculate target costs
   └─> Generate migration plan

4. User Review & Approval
   ├─> Show cost comparison
   ├─> Show timeline estimate
   └─> Show risks and mitigation

5. Parallel Deployment
   ├─> Retrieve original infrar SDK code
   ├─> Transform for target provider
   ├─> Build new Docker image
   ├─> Provision target infrastructure
   └─> Deploy application (parallel to existing)

6. Data Migration (if applicable)
   ├─> Export from source
   ├─> Transform schema
   └─> Import to target

7. Testing Phase
   ├─> Run health checks
   ├─> Validate functionality
   └─> Performance testing

8. DNS Cutover
   ├─> Update DNS records (blue-green)
   ├─> Monitor traffic split
   └─> Gradually shift traffic

9. Monitoring Phase
   ├─> Watch for errors
   ├─> Compare performance
   └─> Validate costs

10. Cleanup (after stability confirmed)
    ├─> Destroy source infrastructure
    ├─> Remove old DNS entries
    └─> Archive old configs
```

## Scalability Considerations

### Horizontal Scaling

- **API Gateway**: Stateless, can run multiple instances behind load balancer
- **Transformation Workers**: Queue-based, add workers as needed
- **Build Workers**: Containerized, run on Kubernetes/ECS
- **Database**: PostgreSQL with read replicas for queries

### Async Processing

- **Build Pipeline**: Queue-based (Redis/RabbitMQ)
- **Deployments**: Long-running tasks in background workers
- **Migrations**: Multi-phase with checkpoints

### Caching

- **Pricing Data**: Cache with TTL (refresh daily)
- **Capability Catalog**: In-memory cache
- **Transformation Rules**: Compiled and cached

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────┐
│ Layer 1: Network Security          │
│ - TLS everywhere                    │
│ - Firewall rules                    │
│ - DDoS protection                   │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ Layer 2: Authentication             │
│ - OAuth 2.0                         │
│ - MFA support                       │
│ - Session management                │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ Layer 3: Authorization              │
│ - RBAC (Role-Based Access Control)  │
│ - Project-level permissions         │
│ - Resource-level policies           │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ Layer 4: Data Protection            │
│ - Encryption at rest                │
│ - Encryption in transit             │
│ - Secrets management (Vault)        │
└─────────────────────────────────────┘
          ↓
┌─────────────────────────────────────┐
│ Layer 5: Audit & Monitoring         │
│ - Access logs                       │
│ - Action audit trail                │
│ - Anomaly detection                 │
└─────────────────────────────────────┘
```

### Credential Isolation

Each customer's cloud credentials are:
- Encrypted with unique key
- Stored in secure vault
- Never logged or exposed
- Used only for specific deployments
- Audited on every access

See: [Security Details](../technical/security.md)

## Reliability & Disaster Recovery

### High Availability

- **Database**: Multi-AZ PostgreSQL with automated backups
- **API**: Multi-region deployment with failover
- **State Files**: Replicated across regions
- **Critical Data**: Real-time replication

### Backup Strategy

- **Database**: Daily automated backups, 30-day retention
- **OpenTofu State**: Versioned object storage
- **Source Code**: Stored in Infrar + user's GitHub
- **Configurations**: Immutable, versioned

### Disaster Recovery

- **RTO (Recovery Time Objective)**: 1 hour
- **RPO (Recovery Point Objective)**: 5 minutes
- **Procedure**:
  1. Failover to backup region
  2. Restore database from latest backup
  3. Verify state file integrity
  4. Resume operations

## Monitoring & Observability

### Metrics

- **System Metrics**: CPU, memory, disk, network
- **Application Metrics**: Request rate, error rate, latency
- **Business Metrics**: Deployments/day, migrations/week, cost savings

### Logging

- **Centralized Logging**: All logs aggregated (CloudWatch, Stackdriver, etc.)
- **Structured Logging**: JSON format for parsing
- **Log Retention**: 90 days standard, 1 year for audit logs

### Tracing

- **Distributed Tracing**: OpenTelemetry for request flows
- **Performance Monitoring**: Identify bottlenecks

### Alerting

- **Critical**: System failures, security incidents
- **Warning**: High error rates, slow responses
- **Info**: Deployment completions, migrations started

## Technology Choices

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| API Backend | Go (Gin) | High performance, great concurrency, strong typing |
| Frontend | React | Rich ecosystem, component-based, widely adopted |
| Database | PostgreSQL | Reliable, JSONB support, full-text search, mature |
| Message Queue | Redis | Fast, simple, pub/sub + queue capabilities |
| Container Orchestration | Kubernetes/ECS | Scalable, self-healing, widely supported |
| IaC Engine | OpenTofu | Open-source, provider-agnostic, Terraform-compatible |
| AST Parsing | Python: ast, Node.js: @babel/parser | Language-native, battle-tested |
| Secrets Management | HashiCorp Vault | Industry standard, audit trails, rotation |

## Next Steps

1. Review [Repository Structure](repository-structure.md) for codebase organization and repository details
2. Review [Transformation Engine](transformation-engine.md) for code transformation details
3. Review [Capability Catalog](capability-catalog.md) for capability definitions
4. Review [Multi-Cloud Orchestration](multi-cloud-orchestration.md) for cross-cloud communication
5. See [MVP Phase 1](../mvp/phase-1.md) for initial implementation plan
