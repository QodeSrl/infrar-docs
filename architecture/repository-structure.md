# Infrar Repository Structure

**Document Status**: Active
**Last Updated**: October 2025
**Organization**: [QodeSrl on GitHub](https://github.com/QodeSrl)

## Overview

Infrar is built as a distributed system across multiple repositories following an **open core model**. This document describes each repository, its purpose, scope, and relationships with other components.

**Total Repositories**: 8 (6 public open source, 2 private proprietary)

## Repository Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Open Source (Public)                      │
│                      Apache 2.0 License                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐    ┌──────────────┐   ┌─────────────┐     │
│  │ infrar-cli  │───>│ infrar-engine│<──│infrar-plugins│     │
│  └─────────────┘    └──────────────┘   └─────────────┘     │
│         │                  ▲                                 │
│         │                  │                                 │
│         ▼                  │                                 │
│  ┌─────────────────┐      │                                 │
│  │ infrar-sdk-     │──────┘                                 │
│  │    python       │                                         │
│  └─────────────────┘                                         │
│                                                               │
│  ┌─────────────────────────────────────────────┐            │
│  │         infrar-docs (documentation)         │            │
│  └─────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  Proprietary (Private)                       │
│                   Commercial License                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────┐         ┌─────────────┐              │
│  │  infrar-platform │<───────>│ infrar-web  │              │
│  │  (Backend API)   │         │ (Frontend)  │              │
│  └──────────────────┘         └─────────────┘              │
│          │                                                   │
│          └──────> Uses infrar-engine for transformations    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Public Repositories (Open Source)

### 1. infrar-engine ⭐ Core Component

**Repository**: [`QodeSrl/infrar-engine`](https://github.com/QodeSrl/infrar-engine)
**Status**: ✅ Created
**License**: Apache License 2.0
**Language**: Go (recommended)
**Visibility**: Public

#### Purpose
The heart of Infrar - contains the core transformation engine that converts provider-agnostic code into native cloud provider SDK code.

#### Scope & Responsibilities
- **AST Parsing**: Parse source code into Abstract Syntax Trees
- **Code Analysis**: Identify Infrar SDK usage patterns
- **Transformation Engine**: Apply transformation rules to generate provider-specific code
- **Plugin System**: Load and manage provider plugins
- **Capability Framework**: Define and manage infrastructure capabilities
- **Code Generation**: Output native SDK code (boto3, google-cloud, azure-sdk)
- **Validation**: Ensure transformed code is syntactically correct

#### Key Components
```
infrar-engine/
├── parser/           # AST parsing for Python, Node.js, Go
├── transformer/      # Core transformation logic
├── generator/        # Code generation for each target
├── plugin/           # Plugin system and loader
├── capability/       # Capability definitions and registry
├── validator/        # Syntax and semantic validation
└── cli/              # Library interface for CLI usage
```

#### Dependencies
- **Consumes**: `infrar-plugins` (transformation rules)
- **Used By**: `infrar-cli`, `infrar-platform`, language SDKs (indirectly)

#### Inputs
- Infrar SDK source code (Python, Node.js, etc.)
- Target cloud provider (AWS, GCP, Azure)
- Transformation rules from plugins
- Configuration (capability mappings, provider settings)

#### Outputs
- Native cloud provider SDK code
- Transformation report (what changed, why)
- Warnings and errors
- Metadata for deployment pipeline

#### Example Flow
```
Input:  infrar.storage.upload(bucket='data', file='report.csv')
        ↓ (AST Parse)
        ↓ (Identify pattern: storage.upload)
        ↓ (Load plugin: infrar-plugins/storage/aws)
        ↓ (Apply transformation rule)
        ↓ (Generate code)
Output: boto3.client('s3').upload_file('report.csv', 'data', 'report.csv')
```

#### Version Strategy
- Semantic versioning (SemVer)
- Stable 1.0.0 when transformation rules solidify
- Breaking changes only on major versions

---

### 2. infrar-sdk-python

**Repository**: [`QodeSrl/infrar-sdk-python`](https://github.com/QodeSrl/infrar-sdk-python)
**Status**: ✅ Created
**License**: Apache License 2.0
**Language**: Python
**Visibility**: Public

#### Purpose
Provider-agnostic Python SDK that developers import in their applications. This is what users write code against.

#### Scope & Responsibilities
- **API Surface**: Define clean, intuitive APIs for cloud operations
- **Type Hints**: Provide full type annotations for IDE support
- **Documentation**: Inline docstrings for all functions
- **Validation**: Input validation before transformation
- **Markers**: Special annotations for transformation engine hints
- **Common Utilities**: Shared helpers (logging, error handling)

#### Key Components
```
infrar-sdk-python/
├── infrar/
│   ├── __init__.py
│   ├── storage/          # Object storage operations
│   │   ├── __init__.py
│   │   ├── upload.py
│   │   ├── download.py
│   │   └── list.py
│   ├── compute/          # Compute operations (Phase 2)
│   ├── database/         # Database operations (Phase 2)
│   ├── messaging/        # Queue/pub-sub (Phase 2)
│   ├── common/           # Shared utilities
│   └── exceptions.py     # Custom exceptions
├── tests/
├── examples/
├── setup.py
├── pyproject.toml
└── README.md
```

#### Example API
```python
from infrar.storage import upload, download, list_objects, delete

# Upload a file
upload(
    bucket='my-data-bucket',
    source='local/report.csv',
    destination='reports/2024/report.csv'
)

# Download a file
download(
    bucket='my-data-bucket',
    source='reports/2024/report.csv',
    destination='local/downloaded.csv'
)

# List objects
objects = list_objects(bucket='my-data-bucket', prefix='reports/')
```

#### Dependencies
- **Consumed By**: User applications
- **Transformed By**: `infrar-engine`
- **No runtime dependencies** on other Infrar components (pure SDK)

#### Distribution
- PyPI package: `pip install infrar`
- Versioned releases
- Python 3.8+ support

#### Design Principles
1. **Simple**: Easy to learn, intuitive APIs
2. **Type-safe**: Full type hints for IDE autocomplete
3. **Provider-agnostic**: No AWS/GCP/Azure specifics in main APIs
4. **Well-documented**: Comprehensive docstrings
5. **Testable**: Easy to mock for unit tests

---

### 3. infrar-plugins

**Repository**: [`QodeSrl/infrar-plugins`](https://github.com/QodeSrl/infrar-plugins)
**Status**: ✅ Created
**License**: Apache License 2.0
**Language**: YAML/JSON (transformation rules) + Go (plugin logic)
**Visibility**: Public

#### Purpose
Contains transformation rules and provider-specific implementations for each cloud provider. This is the "knowledge base" of how to map Infrar SDK calls to native provider SDKs.

#### Scope & Responsibilities
- **Transformation Rules**: Define how each Infrar operation maps to provider operations
- **Provider Implementations**: Code templates for AWS, GCP, Azure
- **Capability Definitions**: Define infrastructure capabilities per provider
- **Service Mappings**: Map abstract capabilities to specific cloud services
- **Cost Metadata**: Pricing hints for cost intelligence (basic info only)

#### Key Components
```
infrar-plugins/
├── packages/
│   ├── storage/
│   │   ├── capability.yaml      # Storage capability definition
│   │   ├── aws/
│   │   │   ├── rules.yaml       # Transformation rules for AWS S3
│   │   │   ├── templates/       # Code templates
│   │   │   └── metadata.yaml    # Service info, regions, etc.
│   │   ├── gcp/
│   │   │   ├── rules.yaml       # Transformation rules for GCS
│   │   │   └── templates/
│   │   └── azure/
│   │       ├── rules.yaml       # Transformation rules for Blob Storage
│   │       └── templates/
│   ├── compute/                  # Phase 2
│   ├── database/                 # Phase 2
│   └── messaging/                # Phase 2
├── shared/
│   └── templates/                # Shared code templates
└── README.md
```

#### Example Transformation Rule (storage/aws/rules.yaml)
```yaml
# Transformation rule for infrar.storage.upload -> boto3
operations:
  - name: upload
    pattern: "infrar.storage.upload"
    target:
      provider: aws
      service: s3
      operation: upload_file

    transformation:
      imports:
        - "import boto3"

      code_template: |
        s3 = boto3.client('s3')
        s3.upload_file(
          '{{ source }}',
          '{{ bucket }}',
          '{{ destination }}'
        )

      parameter_mapping:
        bucket: bucket
        source: Filename
        destination: Key

    requirements:
      - package: boto3
        version: ">=1.28.0"
```

#### Dependencies
- **Used By**: `infrar-engine` (loads these rules)
- **Community Contributions**: Expected to grow with community input

#### Plugin Development
Community members can contribute:
- New cloud providers (DigitalOcean, Linode, etc.)
- New capabilities (caching, CDN, etc.)
- Improved transformation rules
- Provider-specific optimizations

#### Version Strategy
- Plugins versioned together (monorepo)
- Breaking changes coordinated with engine releases
- Per-provider version compatibility matrix

---

### 4. infrar-cli

**Repository**: [`QodeSrl/infrar-cli`](https://github.com/QodeSrl/infrar-cli)
**Status**: ✅ Created
**License**: Apache License 2.0
**Language**: Go (recommended for CLIs)
**Visibility**: Public

#### Purpose
Command-line interface for developers to interact with Infrar locally - initialize projects, transform code, test, and deploy.

#### Scope & Responsibilities
- **Project Initialization**: `infrar init` - scaffold new projects
- **Local Transformation**: Transform code locally for testing
- **Provider Management**: Configure AWS/GCP/Azure credentials
- **Deployment**: Deploy transformed code to cloud providers
- **Cost Estimation**: Basic cost comparison (uses platform API if available)
- **Configuration Management**: Manage infrar.yaml config files
- **Plugin Management**: Install and update plugins

#### Key Components
```
infrar-cli/
├── cmd/
│   ├── init.go           # infrar init
│   ├── transform.go      # infrar transform
│   ├── deploy.go         # infrar deploy
│   ├── cost.go           # infrar cost
│   └── config.go         # infrar config
├── pkg/
│   ├── project/          # Project management
│   ├── provider/         # Cloud provider integrations
│   ├── transformer/      # Wrapper around infrar-engine
│   └── deployer/         # Deployment orchestration
├── internal/
│   └── ui/               # CLI UI components
└── main.go
```

#### Example Commands
```bash
# Initialize a new Infrar project
infrar init my-app --language python

# Transform code for AWS
infrar transform --provider aws

# Deploy to AWS
infrar deploy --provider aws --region us-east-1

# Compare costs across providers
infrar cost compare

# Configure credentials
infrar config set aws.credentials ~/.aws/credentials
```

#### Configuration File (infrar.yaml)
```yaml
project:
  name: my-app
  language: python

capabilities:
  - name: storage
    type: object-storage
    resources:
      - bucket: my-data-bucket

providers:
  aws:
    region: us-east-1
    account: "123456789"
  gcp:
    project: my-gcp-project
    region: us-central1
```

#### Dependencies
- **Uses**: `infrar-engine` (as Go library)
- **Loads**: `infrar-plugins`
- **Optionally Uses**: `infrar-platform` API (for hosted features)

#### Distribution
- Homebrew: `brew install infrar`
- npm: `npm install -g infrar-cli`
- Direct download: Binary releases on GitHub
- Docker: `docker run infrar/cli`

---

### 5. infrar-docs

**Repository**: [`QodeSrl/infrar-docs`](https://github.com/QodeSrl/infrar-docs)
**Status**: ✅ Created
**License**: Apache License 2.0
**Language**: Markdown
**Visibility**: Public

#### Purpose
Comprehensive documentation for Infrar - user guides, API references, architecture docs, tutorials, and examples.

#### Scope & Responsibilities
- **Getting Started**: Quick start guides, tutorials
- **Concepts**: Explain core concepts (capabilities, transformation, etc.)
- **API Reference**: Complete SDK API documentation
- **Architecture**: System design and technical deep-dives
- **User Guides**: How-to guides for common scenarios
- **Examples**: Sample projects and code snippets
- **Contributing**: Guidelines for contributors
- **Changelog**: Release notes and migration guides

#### Structure
```
infrar-docs/
├── getting-started/
│   ├── installation.md
│   ├── quickstart.md
│   └── first-deployment.md
├── concepts/
│   ├── capabilities.md
│   ├── transformation.md
│   ├── plugins.md
│   └── multi-cloud.md
├── guides/
│   ├── python-sdk.md
│   ├── nodejs-sdk.md
│   ├── cli-usage.md
│   └── deployment.md
├── api-reference/
│   ├── storage.md
│   ├── compute.md
│   └── database.md
├── architecture/
│   ├── overview.md
│   ├── transformation-engine.md
│   └── repository-structure.md
├── examples/
│   ├── simple-storage-app/
│   ├── multi-cloud-web-app/
│   └── data-pipeline/
├── contributing/
│   ├── how-to-contribute.md
│   ├── plugin-development.md
│   └── code-style.md
└── README.md
```

#### Documentation Website
- **Technology**: Docusaurus, VitePress, or MkDocs
- **Hosting**: GitHub Pages, Vercel, or Netlify
- **Domain**: docs.infrar.io or infrar.qode.com/docs
- **Search**: Algolia DocSearch
- **Versioning**: Per-release documentation versions

#### Content Sources
- Current docs in `/home/alexb/projects/infrar/docs/` → migrate here
- API docs: Auto-generated from SDK docstrings
- Examples: Linked from example repositories
- Architecture: Technical design documents

---

### 6. .github (Organization Profile)

**Repository**: [`QodeSrl/.github`](https://github.com/QodeSrl/.github)
**Status**: ✅ Created
**License**: MIT
**Language**: Markdown
**Visibility**: Public

#### Purpose
Organization-wide community health files, templates, and the QodeSrl organization profile displayed on github.com/QodeSrl.

#### Scope & Responsibilities
- **Organization Homepage**: profile/README.md displayed on org page
- **Community Guidelines**: Default CONTRIBUTING.md, CODE_OF_CONDUCT.md
- **Security Policy**: SECURITY.md for vulnerability reporting
- **Issue Templates**: Standardized issue/PR templates
- **GitHub Actions**: Reusable workflows

#### Structure
```
.github/
├── profile/
│   └── README.md              # Organization homepage
├── CONTRIBUTING.md            # Default contributing guide
├── CODE_OF_CONDUCT.md         # Code of conduct
├── SECURITY.md                # Security policy
├── ISSUE_TEMPLATE/
│   ├── bug_report.md
│   ├── feature_request.md
│   └── plugin_proposal.md
├── PULL_REQUEST_TEMPLATE.md
└── workflows/                 # Reusable workflows (future)
    └── test.yml
```

#### Organization Profile Content
- Overview of QodeSrl and projects
- Infrar project showcase with links to repos
- Getting started guide
- Community links

---

## Private Repositories (Proprietary)

### 7. infrar-platform 🔒

**Repository**: `QodeSrl/infrar-platform` (to be created)
**Status**: 📋 Planned - Create in Phase 3
**License**: Proprietary (All Rights Reserved)
**Language**: Go (backend services)
**Visibility**: Private

#### Purpose
Backend platform services for the hosted Infrar SaaS offering. Provides intelligence features, brownfield discovery, migration planning, and multi-account management.

#### Scope & Responsibilities
- **API Gateway**: REST/GraphQL API for web UI and CLI
- **Cost Intelligence Engine**: Real-time cost comparison across providers
- **Brownfield Discovery**: Scan and analyze existing cloud infrastructure
- **Migration Planning**: ROI analysis, risk assessment, migration strategies
- **Multi-Cloud Orchestration**: Coordinate deployments across multiple providers
- **User Management**: Authentication, authorization, team management
- **Deployment Pipeline**: Integrate with infrar-engine for hosted transformations
- **Billing & Metering**: Usage tracking and subscription management

#### Key Services
```
infrar-platform/
├── api/                      # REST/GraphQL API gateway
│   ├── handlers/
│   ├── middleware/
│   └── schema/
├── services/
│   ├── cost-intelligence/   # Cost comparison engine
│   │   ├── pricing-db/      # Cloud pricing database
│   │   ├── calculator/      # Cost calculations
│   │   └── optimizer/       # Optimization recommendations
│   ├── discovery/           # Brownfield discovery service
│   │   ├── scanners/        # Cloud account scanners
│   │   ├── mapper/          # Map resources to capabilities
│   │   └── analyzer/        # Analyze code coupling
│   ├── migration/           # Migration planning service
│   │   ├── planner/         # Generate migration plans
│   │   ├── roi/             # ROI calculator
│   │   └── simulator/       # Simulate migration scenarios
│   ├── orchestrator/        # Deployment orchestration
│   │   ├── pipeline/        # CI/CD pipeline management
│   │   ├── builder/         # Container builds
│   │   └── deployer/        # Multi-cloud deployment
│   ├── auth/                # Authentication service
│   └── billing/             # Billing and metering
├── workers/                 # Background job workers
│   ├── scanner-worker/
│   ├── deployment-worker/
│   └── pricing-sync-worker/
├── pkg/                     # Shared packages
│   ├── cloud/               # Cloud provider clients
│   ├── database/            # Database utilities
│   └── transformer/         # Wrapper for infrar-engine
├── infrastructure/          # IaC for platform itself
│   └── terraform/
└── docker-compose.yml       # Local development
```

#### Database Schema (PostgreSQL)
```
Tables:
- users                    # User accounts
- organizations            # Teams/organizations
- projects                 # Infrar projects
- deployments              # Deployment history
- cloud_accounts           # Connected AWS/GCP/Azure accounts
- discovered_resources     # Brownfield discovery results
- capabilities             # User-defined capabilities
- cost_estimates           # Cost comparison results
- migration_plans          # Saved migration plans
- pricing_data             # Cloud provider pricing (updated daily)
- usage_metrics            # Billing and metering
```

#### External Integrations
- **Cloud Providers**: AWS, GCP, Azure APIs (read-only for discovery)
- **GitHub/GitLab**: OAuth, webhook integration for CI/CD
- **Stripe**: Payment processing
- **SendGrid**: Email notifications
- **Datadog**: Monitoring and logging
- **Auth0/Okta**: Enterprise SSO

#### API Examples
```
POST   /api/v1/projects                  # Create project
GET    /api/v1/projects/:id/deployments  # List deployments
POST   /api/v1/transform                 # Transform code
POST   /api/v1/deploy                    # Deploy to provider
GET    /api/v1/cost/compare              # Cost comparison
POST   /api/v1/discovery/scan            # Scan cloud account
GET    /api/v1/migration/plan            # Get migration plan
```

#### Dependencies
- **Uses**: `infrar-engine` (for transformations)
- **Uses**: `infrar-plugins` (for transformation rules)
- **Serves**: `infrar-web` (frontend)
- **Integrates**: `infrar-cli` (optional hosted features)

#### Technology Stack
- **Language**: Go (performance, concurrency)
- **API**: GraphQL (flexibility) or REST
- **Database**: PostgreSQL (relational data, JSON support)
- **Cache**: Redis (session, rate limiting)
- **Queue**: RabbitMQ or AWS SQS (background jobs)
- **Storage**: S3/GCS (transformed code, artifacts)
- **Container Registry**: ECR/Artifact Registry/ACR
- **Observability**: OpenTelemetry, Prometheus, Grafana

#### Deployment
- **Container**: Docker, Kubernetes (EKS, GKE, or AKS)
- **CI/CD**: GitHub Actions
- **IaC**: Terraform/OpenTofu for platform infrastructure
- **Secrets**: AWS Secrets Manager, Vault

---

### 8. infrar-web 🔒

**Repository**: `QodeSrl/infrar-web` (to be created)
**Status**: 📋 Planned - Create in Phase 3
**License**: Proprietary (All Rights Reserved)
**Language**: TypeScript (React/Next.js)
**Visibility**: Private

#### Purpose
Web-based user interface for the Infrar platform. Provides visual tools for project management, cost comparison, brownfield discovery, and migration planning.

#### Scope & Responsibilities
- **Dashboard**: Overview of projects, deployments, costs
- **Project Management**: Create, configure, and manage projects
- **Cost Intelligence UI**: Visual cost comparison across providers
- **Brownfield Discovery UI**: Visualize discovered infrastructure
- **Migration Planning UI**: Interactive migration plan builder
- **Deployment Interface**: Trigger and monitor deployments
- **Team Management**: User/team administration, RBAC
- **Account Settings**: Profile, billing, integrations

#### Key Features
```
infrar-web/
├── src/
│   ├── app/                    # Next.js app directory
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── signup/
│   │   ├── (dashboard)/
│   │   │   ├── projects/
│   │   │   ├── deployments/
│   │   │   ├── cost/
│   │   │   ├── discovery/
│   │   │   └── migration/
│   │   └── settings/
│   ├── components/
│   │   ├── ui/               # Shared UI components
│   │   ├── charts/           # Cost charts, graphs
│   │   ├── editors/          # Code editors (Monaco)
│   │   └── forms/
│   ├── lib/
│   │   ├── api/              # API client for infrar-platform
│   │   ├── auth/             # Authentication utilities
│   │   └── hooks/            # React hooks
│   ├── types/                # TypeScript types
│   └── styles/               # Global styles
├── public/
├── package.json
└── next.config.js
```

#### Key Screens

**1. Dashboard**
- Project overview cards
- Recent deployments
- Cost trends chart
- Quick actions (new project, deploy)

**2. Projects**
- List all projects
- Project details (config, capabilities, providers)
- Edit project configuration
- View transformation results

**3. Cost Comparison**
- Side-by-side provider cost comparison
- Interactive cost calculator
- Filters by capability, region, service
- Charts and visualizations
- Export to CSV/PDF

**4. Brownfield Discovery**
- Connect cloud accounts
- View discovered resources
- Resource→Capability mapping visualization
- Code coupling analysis
- Migration difficulty scores
- Cost savings opportunities

**5. Migration Planning**
- Interactive migration plan builder
- Drag-and-drop capabilities to providers
- ROI calculator with timeline
- Risk assessment matrix
- Dependency graph visualization
- Export migration plan

**6. Deployments**
- Deployment history table
- Real-time deployment logs
- Rollback interface
- Deployment comparison (before/after)

**7. Team Management**
- User list with roles
- Invite team members
- RBAC configuration
- Activity audit log

**8. Settings**
- Profile management
- Cloud provider credentials
- GitHub/GitLab integration
- Billing and subscription
- API keys

#### Technology Stack
- **Framework**: Next.js 14+ (App Router)
- **Language**: TypeScript
- **UI Library**: React 18+
- **Styling**: Tailwind CSS + shadcn/ui
- **State**: React Query (server state) + Zustand (client state)
- **Forms**: React Hook Form + Zod validation
- **Charts**: Recharts or Chart.js
- **Code Editor**: Monaco Editor (VS Code editor)
- **Tables**: TanStack Table
- **Auth**: NextAuth.js or Auth0
- **API**: GraphQL (Apollo Client) or REST (fetch/axios)

#### Design System
- **Component Library**: shadcn/ui (customizable, accessible)
- **Icons**: Lucide Icons
- **Fonts**: Inter (sans-serif)
- **Colors**: Tailwind color palette
- **Dark Mode**: System preference + manual toggle

#### Dependencies
- **API Client**: Connects to `infrar-platform` API
- **Authentication**: OAuth with infrar-platform
- **Real-time Updates**: WebSockets for deployment logs

#### Deployment
- **Hosting**: Vercel (recommended for Next.js)
- **Alternative**: AWS Amplify, Netlify, or self-hosted
- **CI/CD**: GitHub Actions
- **Environment**: Production, staging, development
- **CDN**: Cloudflare or provider CDN

---

## Repository Relationships

### Dependency Graph

```
┌─────────────────────────────────────────────────────────┐
│                    Development Flow                      │
└─────────────────────────────────────────────────────────┘

Developer writes code
         ↓
    infrar-sdk-python (imports)
         ↓
    infrar-cli transform (uses)
         ↓
    infrar-engine (loads)
         ↓
    infrar-plugins (applies rules)
         ↓
    Native provider code (generated)
         ↓
    Deploy to AWS/GCP/Azure

┌─────────────────────────────────────────────────────────┐
│                    Platform Flow                         │
└─────────────────────────────────────────────────────────┘

User opens infrar-web
         ↓
    infrar-web (calls API)
         ↓
    infrar-platform (processes)
         ├──> infrar-engine (transform)
         ├──> infrar-plugins (rules)
         ├──> Cost Intelligence (analyze)
         └──> Discovery (scan)
         ↓
    Results displayed in UI
```

### Build Dependencies

| Repository | Depends On | Used By |
|------------|-----------|----------|
| `infrar-engine` | `infrar-plugins` | `infrar-cli`, `infrar-platform` |
| `infrar-sdk-python` | - | User applications |
| `infrar-plugins` | - | `infrar-engine` |
| `infrar-cli` | `infrar-engine`, `infrar-plugins` | Developers |
| `infrar-docs` | - | Everyone (reference) |
| `infrar-platform` | `infrar-engine`, `infrar-plugins` | `infrar-web`, `infrar-cli` |
| `infrar-web` | `infrar-platform` (API) | End users |

---

## Development Workflow

### Phase 1: MVP (Current Phase)

**Timeline**: Months 1-3
**Focus**: Prove transformation works end-to-end

**Active Repositories**:
1. ✅ `infrar-engine` - Build core transformation logic
2. ✅ `infrar-sdk-python` - Define storage APIs
3. ✅ `infrar-plugins` - Create AWS + GCP storage rules
4. ✅ `infrar-cli` - Basic commands (init, transform, deploy)
5. ✅ `infrar-docs` - Getting started guides

**Milestone**: Successfully transform and deploy a simple Python storage app to AWS and GCP

---

### Phase 2: Expand Capabilities

**Timeline**: Months 4-6
**Focus**: Add more services and language support

**Active Repositories**:
1. `infrar-engine` - Add Node.js support
2. `infrar-sdk-nodejs` - New repo for Node.js SDK
3. `infrar-plugins` - Add compute, database, messaging plugins
4. `infrar-cli` - Enhanced deployment features
5. `infrar-docs` - Comprehensive API reference

**Milestone**: Multi-capability apps running on 3 cloud providers

---

### Phase 3: Intelligence Layer

**Timeline**: Months 7-9
**Focus**: Build platform and intelligence features

**Active Repositories**:
1. 🔒 `infrar-platform` - **Create this repo**
2. 🔒 `infrar-web` - **Create this repo**
3. `infrar-engine` - Platform integration
4. `infrar-cli` - Optional platform features
5. `infrar-docs` - Platform documentation

**Milestone**: Hosted platform with cost intelligence and brownfield discovery

---

## Repository Management

### Versioning Strategy

**Open Source Repos**: Semantic Versioning (SemVer)
- `MAJOR.MINOR.PATCH` (e.g., 1.2.3)
- MAJOR: Breaking changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes

**Version Compatibility Matrix** (example for v1.0):
```
infrar-engine:      v1.0.0
infrar-sdk-python:  v1.0.0 (compatible with engine ^1.0.0)
infrar-plugins:     v1.0.0 (compatible with engine ^1.0.0)
infrar-cli:         v1.0.0 (bundles engine v1.0.0)
```

**Proprietary Repos**: Calendar versioning or internal versioning
- Platform and web can version independently
- Not exposed to public, so more flexible

### Release Process

1. **Development**: Work in feature branches
2. **PR Review**: All changes require PR approval
3. **CI Tests**: Automated testing before merge
4. **Version Bump**: Update version in code
5. **Tag Release**: Create Git tag (e.g., `v1.2.0`)
6. **Build Artifacts**: Generate binaries, packages
7. **Publish**: PyPI, npm, Homebrew, GitHub Releases
8. **Announce**: Changelog, blog post, social media

### Branch Strategy

**Main Branches**:
- `main` - Stable, production-ready code
- `develop` - Integration branch for features

**Feature Branches**:
- `feature/storage-gcp-support`
- `fix/transformation-bug-123`
- `docs/api-reference-update`

**Release Branches** (for long-term support):
- `release/1.x` - Backport critical fixes

### CI/CD Pipelines

**Open Source Repos** (GitHub Actions):
```yaml
# .github/workflows/test.yml
on: [push, pull_request]
jobs:
  test:
    - Run unit tests
    - Run integration tests
    - Check code coverage
    - Lint code

  release:
    - Build artifacts (on tag push)
    - Publish to package registries
    - Create GitHub release
```

**Private Repos**:
- Similar CI/CD but with deployment to staging/prod
- Security scanning (Dependabot, Snyk)
- Container builds and pushes

---

## Security & Access Control

### Repository Access

**Public Repos**:
- Read: Everyone
- Write: QodeSrl organization members only
- Admin: Core team

**Private Repos**:
- Read/Write: QodeSrl organization members only
- Admin: Core team
- Separate teams for backend vs frontend

### Secret Management

**Do NOT commit**:
- Cloud provider credentials
- API keys
- Database passwords
- Private keys

**Use instead**:
- GitHub Secrets (for CI/CD)
- Environment variables (local development)
- Secret management services (production)

### Security Scanning

**Enabled on all repos**:
- Dependabot (dependency updates)
- CodeQL (security analysis)
- Secret scanning (prevent leaked credentials)

---

## Community & Contributions

### Open Source Contributions

**Welcome contributions to**:
- `infrar-engine`: Core improvements, bug fixes
- `infrar-plugins`: New providers, new capabilities
- `infrar-sdk-*`: Bug fixes, API improvements
- `infrar-cli`: New commands, UX improvements
- `infrar-docs`: Documentation improvements, examples

**Contribution Process**:
1. Fork the repository
2. Create feature branch
3. Make changes with tests
4. Submit pull request
5. Code review by maintainers
6. Merge after approval

### Maintainer Guidelines

**Responsibilities**:
- Review PRs within 48 hours
- Maintain code quality standards
- Update documentation
- Triage issues
- Release management
- Community engagement

**Code Review Checklist**:
- ✅ Tests included and passing
- ✅ Documentation updated
- ✅ No breaking changes (or documented)
- ✅ Code style consistent
- ✅ Commit messages clear

---

## Future Repositories (Potential)

As the project grows, we may create additional repositories:

**Ecosystem**:
- `infrar-sdk-go` - Go SDK (Phase 2+)
- `infrar-sdk-java` - Java SDK (Phase 3+)
- `infrar-examples` - Example projects and templates
- `awesome-infrar` - Curated list of resources

**Tooling**:
- `infrar-vscode` - VS Code extension
- `infrar-terraform-provider` - Terraform provider for Infrar
- `infrar-github-action` - GitHub Action for CI/CD

**Infrastructure**:
- `infrar-infrastructure` - IaC for Infrar platform itself
- `infrar-monitoring` - Observability configs

**Community**:
- `infrar-rfcs` - Request for Comments (design proposals)

---

## Monitoring & Metrics

### Repository Health Metrics

**Track per repository**:
- Stars, forks, watchers (community interest)
- Open issues, PR count (activity)
- Contributors (community engagement)
- Release frequency (velocity)
- Code coverage (quality)
- Download stats (adoption)

**Tools**:
- GitHub Insights
- Open source metrics platforms (e.g., CAULDRON.io)

### Success Criteria (Year 1)

**Community**:
- 10K+ stars across all repos
- 100+ external contributors
- 50+ community plugins
- 500+ Discord/Slack members

**Technical**:
- 80%+ test coverage
- <1% transformation error rate
- 3 languages supported (Python, Node.js, Go)
- 3 providers (AWS, GCP, Azure)
- 15+ capabilities defined

**Adoption**:
- 100K+ CLI downloads
- 10K+ SDK installs (PyPI + npm)
- 100+ production deployments

---

## Next Steps

### Immediate Actions (Phase 1 - Weeks 1-4)

1. **infrar-engine**:
   - Design transformation architecture
   - Implement Python AST parser
   - Build basic transformation engine
   - Create plugin loader

2. **infrar-sdk-python**:
   - Define storage module APIs
   - Add type hints and docstrings
   - Write unit tests
   - Publish to TestPyPI

3. **infrar-plugins**:
   - Define storage capability schema
   - Write AWS S3 transformation rules
   - Write GCP GCS transformation rules
   - Create code templates

4. **infrar-cli**:
   - Implement `infrar init` command
   - Implement `infrar transform` command
   - Create sample project templates
   - Test end-to-end flow

5. **infrar-docs**:
   - Write getting started guide
   - Document SDK APIs
   - Create first tutorial
   - Set up documentation website

### Planning for Future (Phase 3)

6. **infrar-platform** (create when ready):
   - Design system architecture
   - Plan database schema
   - Define API contracts
   - Set up infrastructure

7. **infrar-web** (create when ready):
   - Design UI/UX mockups
   - Plan component structure
   - Define user flows
   - Set up design system

---

## Conclusion

This repository structure follows best practices for open core projects:

✅ **Clear separation** between open source and proprietary components
✅ **Logical organization** by concern and responsibility
✅ **Scalable architecture** that can grow with the project
✅ **Community-friendly** structure that encourages contributions
✅ **Maintainable** with clear ownership and dependencies

Each repository has a specific, well-defined purpose, making it easier for contributors to understand where to make changes and for the team to manage the codebase effectively.

---

**Next**: [Start Building the Transformation Engine](../mvp/phase-1.md)

**See Also**:
- [Open Core Strategy](../business/open-core-strategy.md)
- [Architecture Overview](./overview.md)
- [Contributing Guidelines](https://github.com/QodeSrl/.github/CONTRIBUTING.md)
