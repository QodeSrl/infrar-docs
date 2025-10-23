# Architecture Overview

## System Design

Infrar is a multi-cloud infrastructure platform built on a provider-agnostic plugin architecture. The system transforms cloud-agnostic code into native provider implementations and generates infrastructure configurations dynamically from templates.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   User Application Layer                 │
│                                                           │
│  User writes code with Infrar SDK                        │
│  (infrar-sdk-python, infrar-sdk-nodejs, etc.)            │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│                   Platform Layer                         │
│                                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Infrar Platform API (Backend Services)          │   │
│  │  - Project Management                            │   │
│  │  - Transformation Orchestration                  │   │
│  │  - Deployment Pipeline                           │   │
│  │  - Authentication & Authorization                │   │
│  └────────────────────┬─────────────────────────────┘   │
│                       │                                   │
│  ┌────────────────────▼─────────────────────────────┐   │
│  │  Transformation Engine (infrar-engine)           │   │
│  │  - AST Parsing                                   │   │
│  │  - Code Analysis                                 │   │
│  │  - Transformation Rules Application              │   │
│  │  - Native Code Generation                        │   │
│  └────────────────────┬─────────────────────────────┘   │
│                       │                                   │
│                       │ Loads                             │
│                       ↓                                   │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Provider Plugins (infrar-plugins)                │  │
│  │                                                    │  │
│  │  providers/                                       │  │
│  │  ├── aws/          (authentication, services,     │  │
│  │  ├── gcp/           terraform templates,          │  │
│  │  └── azure/         transformation rules)         │  │
│  └────────────────────┬──────────────────────────────┘  │
└───────────────────────┼─────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────┐
│              Cloud Provider Layer                        │
│                                                           │
│  ┌───────────────┐  ┌───────────────┐  ┌────────────┐  │
│  │     AWS       │  │     GCP       │  │   Azure    │  │
│  │               │  │               │  │            │  │
│  │  - S3         │  │  - GCS        │  │  - Blob    │  │
│  │  - Lambda     │  │  - Cloud Run  │  │  - App Svc │  │
│  │  - RDS        │  │  - Cloud SQL  │  │  - SQL DB  │  │
│  └───────────────┘  └───────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Infrar SDK

**Purpose**: Provider-agnostic API for developers

**Languages**: Python (active), Node.js (planned), Go (planned)

**Key Features**:
- Simple, intuitive APIs for cloud operations
- Full type hints and IDE support
- Zero runtime dependencies
- Works in local development (with provider set)

**Example**:
```python
from infrar.storage import upload, download, list_objects

upload(bucket='data', source='report.csv', destination='reports/report.csv')
files = list_objects(bucket='data', prefix='reports/')
```

### 2. Transformation Engine (infrar-engine)

**Purpose**: Convert Infrar SDK calls to native provider SDK calls

**Process**:
1. Parse user code into Abstract Syntax Tree (AST)
2. Identify Infrar SDK usage patterns
3. Load transformation rules from plugins
4. Apply provider-specific transformations
5. Generate native provider code
6. Validate output

**Input**: Code with Infrar SDK
**Output**: Code with native provider SDK (boto3, google-cloud, azure-sdk)

### 3. Provider Plugin System (infrar-plugins)

**Purpose**: Externalize all provider-specific logic

**Structure**:
```
providers/
├── {provider}/
    ├── provider.yaml               # Provider metadata
    ├── regions.yaml                # Available regions
    ├── provider-defaults.yaml      # Default configuration
    ├── pricing-metadata.yaml       # Cost data
    │
    ├── authentication/             # Authentication methods
    │   └── {method}/
    │       ├── method.yaml
    │       ├── schema.yaml
    │       ├── handler.yaml
    │       └── ui-instructions.yaml
    │
    ├── services/                   # Service capabilities
    │   └── {category}/{service}/
    │       ├── service.yaml
    │       ├── defaults.yaml
    │       ├── terraform/main.tf
    │       └── transform/rules.yaml
    │
    ├── terraform-config/           # Terraform templates
    │   ├── provider-block.tf.tmpl
    │   ├── backend.tf.tmpl
    │   └── variables.tf.tmpl
    │
    └── orchestration/              # Bootstrap scripts
        └── bootstrap-script.sh.tmpl
```

**Key Principle**: Zero platform coupling - adding a provider requires no platform code changes

### 4. Infrastructure Generation

**Purpose**: Generate Terraform/OpenTofu configurations from capabilities

**Process**:
1. Detect capabilities from user code
2. Load service terraform modules from plugins
3. Render provider-specific templates
4. Combine into complete terraform configuration
5. Generate tfvars from provider defaults

**Template Engine**: Go text/template with custom functions

### 5. Deployment Pipeline

**Purpose**: Execute infrastructure provisioning and application deployment

**Steps**:
1. Authenticate with provider (using authentication plugin)
2. Run bootstrap script (enable APIs, create state bucket)
3. Execute terraform init/plan/apply
4. Build application container (if needed)
5. Deploy application to provisioned infrastructure
6. Validate deployment
7. Report status

## Data Flow

### Transformation Flow

```
┌──────────────────────┐
│ User Code (Infrar SDK)│
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│   AST Parsing         │  Parse Python/JS/Go code
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│  Capability Detection │  Identify infrar.storage, infrar.compute, etc.
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Load Transform Rules  │  From providers/{provider}/services/{service}/transform/
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│  Apply Transformations│  Replace infrar calls with native SDK calls
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Native Provider Code  │  Output: boto3/google-cloud/azure-sdk code
└──────────────────────┘
```

### Infrastructure Generation Flow

```
┌──────────────────────┐
│ Detected Capabilities │  e.g., [storage, compute]
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Load Service Modules  │  From providers/{provider}/services/{service}/terraform/
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Render Templates      │  provider-block.tf, variables.tf, backend.tf
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Generate tfvars       │  From provider-defaults.yaml
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Complete Terraform    │  Ready for terraform apply
└──────────────────────┘
```

### Deployment Flow

```
┌──────────────────────┐
│ User Triggers Deploy  │  Via platform UI or CLI
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Setup Authentication  │  Load auth plugin, setup credentials
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Run Bootstrap Script  │  Enable APIs, create state bucket
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Execute Terraform     │  init → plan → apply
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Deploy Application    │  Build container, push to registry, deploy
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ Validate & Report     │  Health checks, update status
└──────────────────────┘
```

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Platform Backend** | Go (Gin framework) | Performance, concurrency, strong typing |
| **Transformation Engine** | Go | Fast AST parsing, easy plugin system |
| **Frontend** | React + Next.js | Rich ecosystem, SSR support |
| **Database** | PostgreSQL | Reliability, JSONB support, mature |
| **Template Engine** | Go text/template | Native, powerful, well-documented |
| **IaC** | OpenTofu | Open-source, Terraform-compatible |
| **Container Runtime** | Docker | Industry standard, widely supported |

## Security Architecture

### Authentication

- **Multi-Method Support**: Each provider can have multiple auth methods
- **Plugin-Based**: Authentication logic in plugins, not platform
- **Credential Isolation**: Credentials encrypted at rest, never logged
- **Least Privilege**: Service accounts with minimal required permissions

### Data Protection

- **Encryption at Rest**: AES-256 encryption for stored credentials
- **Encryption in Transit**: TLS 1.3 for all communications
- **Secret Management**: HashiCorp Vault or cloud KMS for production
- **Audit Logging**: All credential access logged

### Infrastructure Security

- **State File Security**: Terraform state encrypted and access-controlled
- **Network Isolation**: Private networks for sensitive workloads
- **IAM Best Practices**: Role-based access, temporary credentials
- **Security Scanning**: Automated scanning for vulnerabilities

## Scalability Considerations

### Horizontal Scaling

- **API Servers**: Stateless, scale behind load balancer
- **Transformation Workers**: Queue-based, add workers as needed
- **Database**: Read replicas for query scaling

### Async Processing

- **Transformation Jobs**: Background queue (Redis/RabbitMQ)
- **Deployments**: Long-running jobs in workers
- **Checkpointing**: Save state for resumable operations

### Caching

- **Provider Metadata**: Cache in-memory, refresh periodically
- **Template Compilation**: Compile once, cache
- **Transformation Rules**: Load once per provider

## Extensibility

### Adding a New Provider

1. Create provider directory in infrar-plugins
2. Add provider metadata files
3. Add authentication method(s)
4. Add service capabilities
5. Add terraform templates
6. Test and submit PR

**No platform code changes required!**

### Adding a New Service

1. Create service directory under existing provider
2. Add service.yaml and defaults.yaml
3. Add terraform/main.tf module
4. Add transform/rules.yaml
5. Test and submit PR

### Adding a New Authentication Method

1. Create auth method directory
2. Add 4 YAML files (method, schema, handler, ui-instructions)
3. Test validation
4. Submit PR

## Design Principles

1. **Provider Agnostic**: Platform code knows nothing about AWS/GCP/Azure specifics
2. **Template Driven**: All code generation uses templates, not string concatenation
3. **Plugin Based**: All provider logic externalized to plugins
4. **Configuration as Code**: Everything defined in YAML, versionable
5. **Zero Coupling**: Adding providers/services requires zero platform changes
6. **Open Core**: Core transformation engine is open source, platform is proprietary

## Next Steps

- See [Provider Plugin System](../concepts/plugins.md) for plugin architecture details
- See [Repository Structure](repository-structure.md) for codebase organization
- See [CONTRIBUTING.md](../CONTRIBUTING.md) for how to add providers
- See [Authentication Plugins](authentication-plugins.md) for auth system details
- See [Template Engine](template-engine.md) for template system details
