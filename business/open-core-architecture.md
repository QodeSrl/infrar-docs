# Infrar Architecture

**Open Core Multi-Cloud Infrastructure Intelligence Platform**

## Overview

Infrar uses an **open core** model where the core platform and transformation engine are open source (GPL v3), while the cost intelligence and web UI are proprietary.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│  infrar-web (Proprietary UI)                           │
│  - User Interface                                       │
│  - Project Management UI                                │
│  - Cost Comparison Dashboard                            │
│  - Deployment Workflow                                  │
└──────────────────┬──────────────────────────────────────┘
                   │ REST API (HTTPS)
                   │
┌──────────────────▼──────────────────────────────────────┐
│  infrar-platform (GPL v3 - Open Source)                │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  API Layer (Gin Framework)                        │ │
│  │  - Auth & JWT                                     │ │
│  │  - Project CRUD                                   │ │
│  │  - File Upload                                    │ │
│  │  - Orchestration                                  │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Service Layer                                    │ │
│  │  - Authentication Service                         │ │
│  │  - Project Service                                │ │
│  │  - Transformation Orchestrator                    │ │
│  │  - Provisioning Service (OpenTofu)                │ │
│  │  - Deployment Service (Docker)                    │ │
│  │  - Cost API Client (HTTP)                         │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Data Layer                                       │ │
│  │  - PostgreSQL (projects, users, deployments)      │ │
│  │  - Redis (cache, sessions)                        │ │
│  └────────────────────────────────────────────────────┘ │
└──────┬────────────────────────┬─────────────────────────┘
       │                        │
       │ Process Execution      │ HTTP API Calls
       │                        │
       ▼                        ▼
┌──────────────────┐  ┌────────────────────────────────┐
│ infrar-engine    │  │ infrar-cost-intelligence      │
│ (GPL v3)         │  │ (Proprietary)                 │
│                  │  │                                │
│ - Code Analysis  │  │ ┌──────────────────────────┐  │
│ - Transformation │  │ │ FastAPI Service          │  │
│ - Plugin System  │  │ │ - Cost Prediction ML     │  │
│ - OpenTofu Gen   │  │ │ - Pricing Database       │  │
└────────┬─────────┘  │ │ - Optimization Engine    │  │
         │            │ └──────────────────────────┘  │
         │            │                                │
         │            │ ┌──────────────────────────┐  │
         ▼            │ │ PostgreSQL               │  │
┌──────────────────┐  │ │ - Pricing data (10M+)    │  │
│ infrar-plugins   │  │ │ - Historical trends      │  │
│ (GPL v3)         │  │ └──────────────────────────┘  │
│                  │  │                                │
│ - AWS Plugin     │  │ ┌──────────────────────────┐  │
│ - GCP Plugin     │  │ │ Redis                    │  │
│ - Azure Plugin   │  │ │ - Prediction cache       │  │
│ - ...more        │  │ └──────────────────────────┘  │
└──────────────────┘  └────────────────────────────────┘
```

## Component Breakdown

### Open Source Components (GPL v3)

#### 1. infrar-platform
**Language**: Go
**License**: GPL v3
**Purpose**: Backend API and orchestration platform

**Responsibilities**:
- User authentication and authorization
- Project and resource management
- File upload and code storage
- Transformation engine orchestration
- OpenTofu/Terraform execution
- Docker container building and deployment
- Integration with cost intelligence API
- Database management (PostgreSQL, Redis)

**Why Open Source**: Users should be able to self-host the platform, audit the code handling their credentials, and contribute improvements. The orchestration logic is not the competitive advantage.

#### 2. infrar-engine
**Language**: Go
**License**: GPL v3
**Purpose**: Code transformation and infrastructure generation

**Responsibilities**:
- Parse and analyze Python code
- Detect required cloud capabilities
- Execute transformation plugins
- Generate OpenTofu/Terraform configurations
- Apply parameterized transformations

**Why Open Source**: The transformation logic is core to the value proposition and should be transparent. Community can contribute plugins and improvements.

#### 3. infrar-sdk-python
**Language**: Python
**License**: GPL v3
**Purpose**: Python SDK for writing cloud-agnostic code

**Responsibilities**:
- Provide abstractions for cloud resources
- HTTP server, database, cache, storage, etc.
- Type hints and IDE support
- Documentation and examples

**Why Open Source**: Developers need to trust and understand the SDK. Open sourcing enables community contributions and adoption.

#### 4. infrar-plugins
**Language**: Go
**License**: GPL v3
**Purpose**: Transformation plugins for each cloud provider

**Responsibilities**:
- AWS transformation logic
- GCP transformation logic
- Azure transformation logic
- Provider-specific resource mapping
- Code generation for each provider

**Why Open Source**: Plugin architecture benefits from community contributions. Users can create custom plugins for their needs.

#### 5. infrar-cli
**Language**: Go
**License**: GPL v3
**Purpose**: Command-line interface

**Responsibilities**:
- Local development and testing
- Project initialization
- Code transformation
- Direct engine interaction

**Why Open Source**: CLI is a developer tool that benefits from transparency and community improvements.

### Proprietary Components

#### 1. infrar-cost-intelligence
**Language**: Python (FastAPI)
**License**: Proprietary
**Purpose**: Cost prediction and pricing intelligence

**Responsibilities**:
- **Pricing Database**: 10M+ pricing records from AWS, GCP, Azure
  - Real-time pricing updates (hourly for compute)
  - Historical price trends (2+ years)
  - Regional pricing variations
  - Reserved instance and spot pricing
  - Volume discounts

- **Machine Learning Predictions**:
  - XGBoost model for cost prediction (95% accuracy)
  - Training on 100K+ real deployments
  - Multi-cloud cost comparison
  - Confidence scoring

- **Optimization Engine**:
  - Multi-objective optimization (NSGA-II)
  - Cost vs performance trade-offs
  - Migration cost analysis
  - Right-sizing recommendations

**Why Proprietary**: This is the competitive advantage. Building and maintaining the pricing database and ML models requires significant investment. The algorithms and data are the core IP.

**Monetization**:
- Hosted SaaS API (pay per call)
- Self-hosted licenses (annual)
- Enterprise custom deployments

#### 2. infrar-web
**Language**: TypeScript/React (Next.js)
**License**: Proprietary
**Purpose**: Web user interface

**Responsibilities**:
- User registration and authentication
- Project management dashboard
- Code upload interface
- Cost comparison visualization
- Deployment configuration wizard
- Resource provisioning UI
- Deployment status monitoring

**Why Proprietary**: The polished UI is a commercial differentiator. Users can self-host the platform and use the API, but the official UI provides the best experience.

**Monetization**:
- Hosted service at infrar.io
- White-label licenses for enterprises
- Custom UI development

## Data Flow: 8-Step MVP Workflow

```
User Action                 Platform                Engine              Cost API
───────────                ─────────               ──────              ────────

1. Register/Login
   ├─→ POST /auth/register
   │   └─→ Store user in DB
   └─→ Return JWT token

2. Upload Code
   ├─→ POST /projects
   │   ├─→ Create project
   │   └─→ Store code in DB
   └─→ POST /projects/:id/code
       └─→ Upload Python file

3. Analyze & Get Costs
   ├─→ POST /projects/:id/transform
   │   ├─→ Call infrar-engine ────→ Analyze code
   │   │                            Detect capabilities
   │   │                            Return: [http_server,
   │   │                                     database, cache]
   │   └─→ Store capabilities
   │
   └─→ GET /projects/:id/deployment-options
       ├─→ Call cost-api.infrar.io ──────────→ POST /predict
       │                                        {capabilities, providers}
       │                                        ├─→ Query pricing DB
       │                                        ├─→ Run ML prediction
       │                                        └─→ Return options
       └─→ Return cost comparison

4. User Selects Provider
   └─→ POST /projects/:id/deployment-config
       └─→ Store: {provider, region, services}

5. Transform Code
   └─→ POST /projects/:id/transform
       ├─→ Call infrar-engine ────→ Transform code
       │                            Apply AWS plugin
       │                            Generate Terraform
       │                            Return transformed code
       └─→ Store transformed code

6. Provide Credentials
   └─→ POST /projects/:id/credentials
       ├─→ Encrypt credentials
       └─→ Store in DB

7. Provision Resources
   └─→ POST /projects/:id/provision
       ├─→ Generate Terraform from template
       ├─→ Execute OpenTofu
       │   ├─→ terraform init
       │   ├─→ terraform plan
       │   └─→ terraform apply
       └─→ Store resource IDs

8. Deploy Application
   └─→ POST /projects/:id/deploy
       ├─→ Build Docker container
       ├─→ Push to registry
       └─→ Deploy to provisioned resources
```

## Open Core Benefits

### For Users

**Self-Hosting Freedom**:
- Deploy infrar-platform on your own infrastructure
- Full control over data and credentials
- No vendor lock-in for orchestration

**Transparency**:
- Audit code handling cloud credentials
- Understand transformation logic
- Verify security practices

**Customization**:
- Fork and modify the platform
- Create custom plugins
- Integrate with internal tools

### For Qode S.r.l.

**Business Model**:
- **Hosted Platform**: SaaS at infrar.io with cost intelligence included
- **Cost API**: Pay-per-call or subscription for self-hosters
- **Enterprise Licenses**: Custom deployments with support
- **Professional Services**: Custom plugin development, integrations

**Competitive Advantages**:
- Proprietary pricing database (10M+ records, constantly updated)
- ML cost prediction models (trained on real data)
- Polished web UI (React/Next.js with best UX)
- Hosted infrastructure (Cloudflare + Hetzner, €18/month)

**Community Benefits**:
- Open source engine attracts contributors
- Plugin ecosystem grows organically
- Bug reports and security audits
- Increased adoption through trust

## Security Model

### Open Source Platform
- **Code Auditing**: Anyone can review for vulnerabilities
- **Credential Handling**: Encrypted at rest, never logged
- **JWT Authentication**: Standard, well-tested approach
- **SQL Injection Protection**: Parameterized queries only

### Proprietary Cost Intelligence
- **API Authentication**: API keys with rate limiting
- **Data Privacy**: Pricing data never shared
- **ML Model Protection**: Models not exposed
- **License Enforcement**: Key validation on each request

## Deployment Options

### Option 1: Fully Hosted (Recommended for Most Users)
```
User → infrar.io (web) → api.infrar.io (platform) → cost-api.infrar.io (cost)
```
- Everything managed by Qode S.r.l.
- No setup required
- Automatic updates
- Included cost intelligence
- **Cost**: Usage-based pricing

### Option 2: Self-Hosted Platform + Hosted Cost API
```
User → Your Frontend → Your infrar-platform → cost-api.infrar.io (cost)
```
- You host platform on your infrastructure
- Use our hosted cost intelligence API
- Data stays on your servers
- **Cost**: API subscription + your infrastructure

### Option 3: Fully Self-Hosted (Enterprise)
```
User → Your Frontend → Your infrar-platform → Your cost-intelligence
```
- Everything on your infrastructure
- Requires cost intelligence license
- Full control and customization
- **Cost**: Annual licenses + your infrastructure

### Option 4: Development (No Cost Intelligence)
```
User → Local Platform → Stub Cost API (returns placeholder data)
```
- For development and testing
- No cost predictions (mock data)
- Free

## Technology Stack

### infrar-platform (Open Source)
- **Language**: Go 1.21+
- **Framework**: Gin (web framework)
- **Database**: PostgreSQL 15
- **Cache**: Redis 7
- **Auth**: JWT (golang-jwt)
- **Container**: Docker
- **IaC**: OpenTofu/Terraform

### infrar-cost-intelligence (Proprietary)
- **Language**: Python 3.11+
- **Framework**: FastAPI
- **Database**: PostgreSQL 15 (pricing data)
- **Cache**: Redis 7
- **ML**: XGBoost, scikit-learn, pandas
- **Optimization**: NSGA-II (multi-objective)

### infrar-web (Proprietary)
- **Framework**: Next.js 14 (App Router)
- **Language**: TypeScript
- **Styling**: Tailwind CSS
- **State**: Zustand
- **Data Fetching**: TanStack Query
- **HTTP**: Axios

### infrar-engine (Open Source)
- **Language**: Go 1.21+
- **Parser**: go/parser, go/ast
- **Plugin System**: Go plugins
- **IaC Generation**: text/template

## Repository Structure

```
infrar/
├── infrar-platform/         # GPL v3 - Backend API
├── infrar-engine/           # GPL v3 - Transformation engine
├── infrar-sdk-python/       # GPL v3 - Python SDK
├── infrar-plugins/          # GPL v3 - Cloud provider plugins
├── infrar-cli/              # GPL v3 - Command-line tool
├── infrar-cost-intelligence/  # Proprietary - Cost API
├── infrar-web/              # Proprietary - Web UI
└── infrar-docs/             # Documentation (mixed)
```

## Contributing

### Open Source Repositories
- Contributions welcome via pull requests
- Follow CONTRIBUTING.md guidelines
- Code of Conduct applies
- GPL v3 license for all contributions

### Proprietary Repositories
- Not open for external contributions
- Bug reports accepted via GitHub issues
- Feature requests considered

## License Summary

| Component | License | Self-Hostable | Modifiable | Commercial Use |
|-----------|---------|---------------|------------|----------------|
| infrar-platform | GPL v3 | ✅ Yes | ✅ Yes | ✅ Yes (with attribution) |
| infrar-engine | GPL v3 | ✅ Yes | ✅ Yes | ✅ Yes (with attribution) |
| infrar-sdk-python | GPL v3 | ✅ Yes | ✅ Yes | ✅ Yes (with attribution) |
| infrar-plugins | GPL v3 | ✅ Yes | ✅ Yes | ✅ Yes (with attribution) |
| infrar-cli | GPL v3 | ✅ Yes | ✅ Yes | ✅ Yes (with attribution) |
| infrar-cost-intelligence | Proprietary | ⚠️  With License | ❌ No | ⚠️  With License |
| infrar-web | Proprietary | ⚠️  With License | ❌ No | ⚠️  With License |

## Questions?

- **General**: info@infrar.io
- **Sales**: sales@infrar.io
- **Support**: support@infrar.io
- **Licensing**: legal@qode.com
- **Open Source**: Contribute via GitHub

---

**© 2025 Qode S.r.l.**
Platform: GPL v3 | Cost Intelligence & Web: Proprietary
