# Infrar MVP Testing Plan

**Date**: October 2025
**Status**: Planning Phase
**Goal**: End-to-end testing of complete Infrar workflow

---

## Table of Contents

1. [Current State Assessment](#current-state-assessment)
2. [MVP Workflow Analysis](#mvp-workflow-analysis)
3. [Component Architecture](#component-architecture)
4. [Missing Components](#missing-components)
5. [MVP Scope Definition](#mvp-scope-definition)
6. [Testing Strategy](#testing-strategy)
7. [Implementation Plan](#implementation-plan)
8. [Risks & Mitigations](#risks--mitigations)

---

## Current State Assessment

### ✅ Existing Components

| Component | Status | Completeness | Notes |
|-----------|--------|--------------|-------|
| **infrar-engine** | ✅ Built | 100% | Transformation engine working, 17/17 tests passing |
| **infrar-sdk-python** | ✅ Built | 100% | Storage module complete (upload, download, delete, list) |
| **infrar-plugins** | ⚠️ Partial | 40% | Storage plugins (AWS S3, GCP GCS) complete. OpenTofu modules exist but untested |
| **infrar-cli** | ✅ Built | 80% | Transform command working, deploy not implemented |
| **infrar-docs** | ✅ Built | 60% | Documentation exists, needs MVP workflow docs |

### ❌ Missing Components

| Component | Status | Priority | Required For |
|-----------|--------|----------|--------------|
| **infrar-web** | ❌ Not Started | P0 | User interface (steps 1-4, 6) |
| **infrar-platform** | ❌ Not Started | P0 | Backend API (all steps) |
| **Cost Intelligence** | ❌ Not Started | P0 | Pricing database & comparison (step 3) |
| **Credential Manager** | ❌ Not Started | P0 | Secure credential storage (step 6) |
| **OpenTofu Executor** | ❌ Not Started | P0 | Resource provisioning (step 7) |
| **Deployment Service** | ❌ Not Started | P0 | Code deployment (step 8) |
| **Container Builder** | ❌ Not Started | P0 | Docker image creation (step 8) |

---

## MVP Workflow Analysis

### Step 1: User Registration & Login

**User Action**: Register account → Verify email → Login

**Technical Requirements**:
```
Frontend (infrar-web):
├── /signup page
├── /login page
└── /dashboard (protected route)

Backend (infrar-platform):
├── POST /api/auth/register
├── POST /api/auth/login
├── POST /api/auth/verify-email
└── JWT token generation

Database:
└── users table (id, email, password_hash, created_at, verified)
```

**Security**:
- Password hashing (bcrypt/argon2)
- Email verification required
- JWT with refresh tokens
- Rate limiting on auth endpoints

**Testing**:
- ✅ Can register new account
- ✅ Email verification required
- ✅ Cannot login with wrong password
- ✅ JWT token works for authenticated routes
- ✅ Session persists across page refresh

---

### Step 2: Upload Code

**User Action**: Upload Python file → Validate syntax → Store temporarily

**Technical Requirements**:
```
Frontend:
├── File upload component (drag & drop)
├── Code editor (Monaco) for review/edit
└── Syntax highlighting

Backend:
├── POST /api/projects (create project)
├── POST /api/projects/:id/code (upload file)
├── Python syntax validation
└── Temporary storage (S3/local)

Database:
└── projects table (id, user_id, name, original_code, created_at)
```

**Validation**:
- Valid Python syntax
- Contains Infrar SDK imports
- File size limit (< 1MB for MVP)
- Detect required capabilities (storage, compute, etc.)

**Testing**:
- ✅ Can upload valid Python file
- ✅ Syntax errors detected and shown
- ✅ Can edit code in browser
- ✅ Detects required capabilities
- ❌ Rejects non-Python files
- ❌ Rejects files without Infrar SDK

---

### Step 3: Show Deployment Possibilities with Cost Preview

**User Action**: View provider options → Compare costs → See resource breakdown

**Technical Requirements**:
```
Backend:
├── Code analyzer (detect resources needed)
├── Capability mapper (code → capabilities)
├── Pricing database (AWS/GCP/Azure)
├── Cost calculator
└── GET /api/projects/:id/deployment-options

Cost Intelligence (Proprietary):
├── Pricing sync service (daily updates)
├── Resource estimation algorithm
├── Cost comparison engine
└── Optimization recommendations

Frontend:
├── Comparison cards (AWS vs GCP vs Azure)
├── Cost breakdown table
├── Resource list (buckets, compute, etc.)
└── Filter by provider/region
```

**Example Response**:
```json
{
  "project_id": "proj_123",
  "detected_capabilities": ["storage"],
  "deployment_options": [
    {
      "provider": "aws",
      "region": "us-east-1",
      "services": {
        "storage": {
          "service": "S3",
          "resources": ["bucket: analytics-data"],
          "monthly_cost": 23.00
        }
      },
      "total_monthly_cost": 23.00,
      "estimated_setup_time": "5 minutes"
    },
    {
      "provider": "gcp",
      "region": "us-central1",
      "services": {
        "storage": {
          "service": "Cloud Storage",
          "resources": ["bucket: analytics-data"],
          "monthly_cost": 20.00
        }
      },
      "total_monthly_cost": 20.00,
      "estimated_setup_time": "5 minutes"
    }
  ],
  "recommendation": "gcp",
  "recommendation_reason": "13% cheaper, similar performance"
}
```

**Pricing Database Schema**:
```sql
CREATE TABLE pricing_data (
  id UUID PRIMARY KEY,
  provider VARCHAR(10), -- aws, gcp, azure
  service VARCHAR(50),  -- s3, gcs, blob
  region VARCHAR(20),
  resource_type VARCHAR(50),
  unit VARCHAR(20),     -- per GB, per request
  price_per_unit DECIMAL(10,6),
  currency VARCHAR(3),  -- USD
  effective_date DATE,
  updated_at TIMESTAMP
);

CREATE INDEX idx_pricing ON pricing_data(provider, service, region);
```

**Testing**:
- ✅ Detects storage capability from code
- ✅ Returns cost estimates for AWS and GCP
- ✅ Shows resource breakdown
- ✅ Recommendation is calculated correctly
- ✅ Pricing data is current (< 7 days old)

---

### Step 4: User Decides Provider & Products

**User Action**: Select provider (AWS/GCP) → Select region → Confirm

**Technical Requirements**:
```
Frontend:
├── Provider selection cards
├── Region dropdown
├── Resource naming inputs
└── Confirmation dialog

Backend:
├── POST /api/projects/:id/deployment-config
└── Store selected configuration

Database:
└── deployment_configs table (
     id, project_id, provider, region,
     resource_names, status, created_at
   )
```

**Resource Naming**:
- User provides names for:
  - Bucket names (must be globally unique)
  - Compute service names
  - Database names
- Validation:
  - Check naming rules per provider
  - Check availability (bucket name not taken)
  - Suggest alternatives if taken

**Testing**:
- ✅ Can select AWS
- ✅ Can select region
- ✅ Can customize resource names
- ✅ Validates bucket name availability
- ✅ Stores configuration correctly

---

### Step 5: Engine Transforms Code (Parametrized)

**User Action**: Click "Transform Code" → View transformed output

**Technical Requirements**:
```
Backend:
├── GET /api/projects/:id/transform
├── Call infrar-engine transformation
├── Parametrize resource names
├── Generate provider-specific code
└── Store transformed code

Transformation Flow:
1. Load user's Python code
2. Load deployment config (provider, resource names)
3. Call infrar-engine with:
   - Source code
   - Target provider (aws/gcp)
   - Capability (storage)
4. Parametrize output:
   - Replace hardcoded bucket names with user's chosen names
   - Add terraform variable references
5. Return transformed code + metadata

Frontend:
├── Show original vs transformed code (split view)
├── Highlight changes
└── Download transformed code button
```

**Example Transformation**:

**Input (user code)**:
```python
from infrar.storage import upload

upload(
    bucket='my-bucket',
    source='data.csv',
    destination='backup/data.csv'
)
```

**Output (transformed for AWS, parametrized)**:
```python
import boto3
import os

# Resource names from Infrar configuration
BUCKET_NAME = os.environ.get('INFRAR_BUCKET_NAME', 'user-chosen-bucket-name')

s3 = boto3.client('s3')
s3.upload_file('data.csv', BUCKET_NAME, 'backup/data.csv')
```

**Testing**:
- ✅ Transforms storage operations correctly
- ✅ Parametrizes bucket names
- ✅ Works for AWS (boto3)
- ✅ Works for GCP (google-cloud-storage)
- ✅ Stores transformed code in database

---

### Step 6: User Creates Provider Account & Provides Credentials

**User Action**: Create AWS/GCP account → Setup billing → Provide credentials

**Technical Requirements**:
```
Frontend:
├── Credentials input form
│   ├── AWS: Access Key ID + Secret Access Key
│   └── GCP: Service Account JSON
├── Credential validation indicator
└── Help text with setup instructions

Backend:
├── POST /api/projects/:id/credentials
├── Validate credentials (test API call)
├── Encrypt credentials
├── Store in secure storage (AWS Secrets Manager/Vault)
└── Never log credentials

Database:
└── cloud_credentials table (
     id, project_id, provider,
     secret_arn, -- ARN to secret in Secrets Manager
     validated, validated_at, created_at
   )
```

**Credential Validation**:
- **AWS**: Call `sts:GetCallerIdentity` to verify credentials work
- **GCP**: Call `storage.buckets.list` to verify service account

**Security Requirements**:
- ✅ Credentials encrypted at rest
- ✅ Credentials encrypted in transit (HTTPS)
- ✅ Never stored in application database
- ✅ Use AWS Secrets Manager or HashiCorp Vault
- ✅ Credentials scoped to minimum permissions
- ✅ Audit log for credential access

**Testing**:
- ✅ Valid AWS credentials accepted
- ✅ Invalid AWS credentials rejected
- ✅ Valid GCP service account accepted
- ✅ Invalid GCP service account rejected
- ✅ Credentials stored securely (not in DB plaintext)
- ✅ Can retrieve credentials for deployment

---

### Step 7: Infrar Creates Resources with OpenTofu

**User Action**: Click "Provision Resources" → Monitor progress → Resources created

**Technical Requirements**:
```
Backend:
├── POST /api/projects/:id/provision
├── OpenTofu executor service
├── Generate Terraform config dynamically
├── Execute terraform apply
├── Stream logs to frontend
└── Store state in provider account

OpenTofu Executor:
├── Generate .tf files from templates
├── Initialize terraform (terraform init)
├── Plan infrastructure (terraform plan)
├── Apply changes (terraform apply -auto-approve)
├── Store state in S3/GCS (in user's account)
└── Track provisioning status

Generated Terraform Example (AWS):
```hcl
terraform {
  backend "s3" {
    bucket = "infrar-tfstate-user-chosen-name"
    key    = "project-123/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  default = "us-east-1"
}

variable "bucket_name" {
  default = "user-chosen-bucket-name"
}

module "storage" {
  source = "./infrar-plugins/packages/storage/aws/terraform"

  bucket_name        = var.bucket_name
  versioning_enabled = true
  tags = {
    ManagedBy = "Infrar"
    ProjectId = "proj_123"
  }
}

output "bucket_arn" {
  value = module.storage.bucket_arn
}
```

**State Management**:
```
State Storage Location:
├── AWS: S3 bucket in user's account
│   └── Bucket: infrar-tfstate-{project_id}
│   └── Key: project-{id}/terraform.tfstate
└── GCP: GCS bucket in user's project
    └── Bucket: infrar-tfstate-{project_id}
    └── Key: project-{id}/terraform.tfstate

Database:
└── terraform_states table (
     id, project_id, provider,
     state_location, -- S3/GCS path
     last_applied_at, status
   )
```

**Provisioning Workflow**:
```
1. Create state bucket (if not exists)
   - AWS: Create S3 bucket for Terraform state
   - GCP: Create GCS bucket for Terraform state

2. Generate Terraform config
   - Load OpenTofu modules from infrar-plugins
   - Parametrize with user's resource names
   - Configure backend (S3/GCS state storage)

3. Initialize Terraform
   - terraform init
   - Download providers (AWS/GCP)
   - Configure backend

4. Plan infrastructure
   - terraform plan -out=plan.tfplan
   - Show resources to be created
   - Estimate costs

5. Apply infrastructure
   - terraform apply plan.tfplan
   - Create resources in user's account
   - Store outputs (bucket ARN, etc.)

6. Store state
   - State automatically saved to S3/GCS
   - Track state location in database
```

**Frontend:**
```
├── Provisioning status page
├── Live logs stream (WebSocket)
├── Progress indicator
│   ├── Creating state bucket...
│   ├── Initializing Terraform...
│   ├── Planning infrastructure...
│   └── Applying changes...
├── Resource list (created resources)
└── Error handling & rollback button
```

**Testing**:
- ✅ State bucket created in user's AWS account
- ✅ Terraform initialized successfully
- ✅ Storage bucket created with correct name
- ✅ State saved to S3
- ✅ Can retrieve state for updates
- ✅ Idempotent (re-running doesn't create duplicates)
- ❌ Handles errors gracefully
- ❌ Can rollback on failure

---

### Step 8: Deploy Code to Target Account

**User Action**: Click "Deploy Code" → Monitor deployment → Application running

**Technical Requirements**:
```
Backend:
├── POST /api/projects/:id/deploy
├── Container builder service
├── Container registry manager
├── Deployment orchestrator
└── Health check service

Deployment Workflow:
1. Create Dockerfile
2. Build Docker image
3. Push to container registry (ECR/GCR)
4. Deploy to compute service
5. Run health checks
6. Update deployment status
```

**Container Build**:
```dockerfile
# Generated Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install boto3>=1.28.0  # or google-cloud-storage for GCP

# Copy transformed code
COPY app.py /app/

# Set environment variables
ENV INFRAR_BUCKET_NAME=${BUCKET_NAME}

# Run application
CMD ["python", "app.py"]
```

**Deployment by Provider**:

**AWS (ECS Fargate)**:
```hcl
# Generated via OpenTofu module
module "compute" {
  source = "./infrar-plugins/packages/compute/aws/terraform"

  app_name        = "user-app-name"
  container_image = "${ECR_URL}:latest"
  container_port  = 8080
  cpu             = 256
  memory          = 512

  environment_variables = {
    INFRAR_BUCKET_NAME = var.bucket_name
  }
}
```

**GCP (Cloud Run)**:
```hcl
# Generated via OpenTofu module
module "compute" {
  source = "./infrar-plugins/packages/compute/gcp/terraform"

  service_name    = "user-app-name"
  container_image = "${GCR_URL}:latest"

  environment_variables = {
    INFRAR_BUCKET_NAME = var.bucket_name
  }
}
```

**Deployment Status Tracking**:
```sql
CREATE TABLE deployments (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  status VARCHAR(20), -- building, pushing, deploying, running, failed
  container_image TEXT,
  deployment_url TEXT,
  logs TEXT,
  created_at TIMESTAMP,
  completed_at TIMESTAMP
);
```

**Frontend**:
```
├── Deployment progress page
├── Build logs (real-time)
├── Deployment status
├── Application URL (when deployed)
└── View logs button
```

**Testing**:
- ✅ Docker image builds successfully
- ✅ Image pushed to ECR/GCR
- ✅ Service deployed to ECS/Cloud Run
- ✅ Application is accessible via URL
- ✅ Environment variables set correctly
- ✅ Can access S3/GCS bucket from application
- ❌ Health checks pass
- ❌ Logs are captured and viewable

---

## Component Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        User Browser                          │
│                      (infrar-web)                            │
│  - Authentication UI                                         │
│  - Code Upload                                               │
│  - Cost Comparison                                           │
│  - Resource Configuration                                    │
│  - Deployment Dashboard                                      │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS/WebSocket
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                   Backend Platform                           │
│                  (infrar-platform)                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  API Gateway (Go/Gin)                                │  │
│  │  - Authentication                                    │  │
│  │  - Request routing                                   │  │
│  │  - Rate limiting                                     │  │
│  └────────────┬─────────────────────────────────────────┘  │
│               │                                              │
│  ┌────────────┴─────────────┬────────────────┬───────────┐ │
│  │                          │                │           │ │
│  ▼                          ▼                ▼           ▼ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐
│  │ Auth Service │  │ Code Service │  │ Cost Service │  │ Deploy   │
│  │              │  │              │  │ (Proprietary)│  │ Service  │
│  │ - Register   │  │ - Upload     │  │ - Pricing DB │  │ - Build  │
│  │ - Login      │  │ - Validate   │  │ - Compare    │  │ - Push   │
│  │ - JWT        │  │ - Transform  │  │ - Recommend  │  │ - Deploy │
│  └──────────────┘  └──────┬───────┘  └──────────────┘  └────┬─────┘
│                            │                                  │
│                            ↓                                  │
│                   ┌─────────────────┐                        │
│                   │ Transformation  │                        │
│                   │ Engine          │                        │
│                   │ (infrar-engine) │                        │
│                   └─────────────────┘                        │
│                            │                                  │
│                            ↓                                  │
│                   ┌─────────────────┐                        │
│                   │ Plugin Loader   │                        │
│                   │ (infrar-plugins)│                        │
│                   └─────────────────┘                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  OpenTofu Executor                                   │  │
│  │  - Generate .tf files                                │  │
│  │  - terraform init/plan/apply                         │  │
│  │  - State management                                  │  │
│  └──────────────────────────────────────────────────────┘  │
└───────────────────────┬──────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ↓               ↓               ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  PostgreSQL  │ │ AWS Secrets  │ │   S3/GCS     │
│  Database    │ │  Manager     │ │ (Infrar      │
│              │ │              │ │  Storage)    │
│ - Users      │ │ - Provider   │ │ - Code files │
│ - Projects   │ │   credentials│ │ - Artifacts  │
│ - Pricing    │ └──────────────┘ └──────────────┘
│ - Deployments│
└──────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────┐
│                   User's Cloud Account                       │
│                    (AWS or GCP)                              │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Terraform    │  │   Storage    │  │   Compute    │     │
│  │ State        │  │   (S3/GCS)   │  │ (ECS/Cloud   │     │
│  │ (S3/GCS)     │  │              │  │  Run)        │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ User's Application (Deployed)                        │  │
│  │ - Containerized app                                  │  │
│  │ - Uses native SDK (boto3/google-cloud-storage)      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Database Schema

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  email_verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Projects
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  original_code TEXT,
  transformed_code TEXT,
  detected_capabilities JSONB, -- ["storage", "compute"]
  status VARCHAR(50), -- uploaded, analyzed, configured, provisioned, deployed
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Deployment Configurations
CREATE TABLE deployment_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  provider VARCHAR(10) NOT NULL, -- aws, gcp, azure
  region VARCHAR(50) NOT NULL,
  resource_names JSONB, -- {"bucket": "user-bucket-name"}
  estimated_cost DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Cloud Credentials (stores reference to secret, not actual credentials)
CREATE TABLE cloud_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  provider VARCHAR(10) NOT NULL,
  secret_arn TEXT NOT NULL, -- ARN/path to secret in Secrets Manager/Vault
  validated BOOLEAN DEFAULT FALSE,
  validated_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Terraform States
CREATE TABLE terraform_states (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  provider VARCHAR(10) NOT NULL,
  state_location TEXT, -- s3://bucket/path or gs://bucket/path
  last_applied_at TIMESTAMP,
  status VARCHAR(50), -- initializing, planning, applying, applied, failed
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Deployments
CREATE TABLE deployments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  status VARCHAR(50), -- building, pushing, deploying, running, failed
  container_image TEXT,
  deployment_url TEXT,
  logs TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);

-- Pricing Data (Proprietary)
CREATE TABLE pricing_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider VARCHAR(10) NOT NULL,
  service VARCHAR(50) NOT NULL,
  region VARCHAR(50) NOT NULL,
  resource_type VARCHAR(50),
  unit VARCHAR(20),
  price_per_unit DECIMAL(10,6),
  currency VARCHAR(3) DEFAULT 'USD',
  effective_date DATE NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(provider, service, region, resource_type, effective_date)
);

CREATE INDEX idx_pricing_lookup ON pricing_data(provider, service, region);
CREATE INDEX idx_pricing_date ON pricing_data(effective_date DESC);
```

---

## Missing Components

### 1. Frontend (infrar-web) - Priority P0

**Technology Stack**:
- Framework: Next.js 14+ (App Router)
- Language: TypeScript
- UI: Tailwind CSS + shadcn/ui
- State: React Query + Zustand
- Auth: NextAuth.js or custom JWT

**Pages Needed**:
```
/                      → Landing page
/signup               → Registration
/login                → Login
/dashboard            → User dashboard
/projects/new         → Create project (upload code)
/projects/:id         → Project details
/projects/:id/deploy  → Deployment options & config
/projects/:id/status  → Deployment status
```

**Estimated Effort**: 2-3 weeks

---

### 2. Backend Platform (infrar-platform) - Priority P0

**Technology Stack**:
- Language: Go 1.21+
- Framework: Gin or Echo
- Database: PostgreSQL 15+
- Cache: Redis (optional for MVP)
- Queue: Built-in Go channels (MVP), later RabbitMQ/SQS

**Services Needed**:
```
infrar-platform/
├── api/
│   ├── auth/          → Registration, login, JWT
│   ├── projects/      → Project CRUD
│   ├── transform/     → Code transformation
│   ├── cost/          → Cost estimation
│   ├── provision/     → Resource provisioning
│   └── deploy/        → Application deployment
├── services/
│   ├── transformer/   → Wrapper for infrar-engine
│   ├── cost/          → Cost intelligence (proprietary)
│   ├── terraform/     → OpenTofu executor
│   └── docker/        → Container builder
└── main.go
```

**Estimated Effort**: 4-5 weeks

---

### 3. Cost Intelligence Engine - Priority P0

**Requirements**:
- Pricing database (AWS/GCP/Azure)
- Daily pricing sync service
- Cost estimation algorithms
- Provider comparison logic

**Data Sources**:
- AWS Pricing API
- GCP Cloud Billing API
- Azure Pricing API

**Estimated Effort**: 2 weeks

---

### 4. OpenTofu Executor - Priority P0

**Requirements**:
- Generate Terraform configs dynamically
- Execute terraform init/plan/apply
- Stream logs to frontend
- Manage state storage

**Estimated Effort**: 1-2 weeks

---

### 5. Container Builder & Deployment - Priority P1

**Requirements**:
- Build Docker images
- Push to ECR/GCR
- Deploy to ECS Fargate or Cloud Run
- Health checks

**Estimated Effort**: 2 weeks

---

## MVP Scope Definition

### Must Have (MVP)

1. ✅ **Authentication**
   - Email/password registration
   - Login with JWT
   - Basic user dashboard

2. ✅ **Code Upload**
   - Upload single Python file
   - Syntax validation
   - Capability detection (storage only)

3. ✅ **Cost Comparison** (Simplified)
   - Show AWS vs GCP costs for storage
   - Hardcoded pricing (update monthly)
   - Basic recommendation logic

4. ✅ **Provider Selection**
   - Choose AWS or GCP
   - Choose region
   - Name resources

5. ✅ **Code Transformation**
   - Transform to boto3 (AWS) or google-cloud-storage (GCP)
   - Parametrize bucket names
   - Show before/after code

6. ✅ **Credentials**
   - Input AWS/GCP credentials
   - Validate credentials
   - Store securely in Secrets Manager

7. ✅ **Resource Provisioning**
   - Create S3/GCS bucket via OpenTofu
   - Store state in user's account
   - Show provisioning logs

8. ⚠️ **Deployment** (Simplified)
   - Manual deployment instructions
   - Provide Dockerfile
   - User deploys themselves (MVP shortcut)

### Nice to Have (Post-MVP)

- Multiple file projects
- Environment variables UI
- Rollback functionality
- Cost tracking dashboard
- Multi-region deployments
- Database capabilities
- Messaging capabilities

---

## Testing Strategy

### Unit Testing

**Backend (Go)**:
```bash
go test ./... -v -cover
```

**Coverage Targets**:
- API handlers: 80%+
- Business logic: 90%+
- Cost calculator: 95%+

**Frontend (TypeScript)**:
```bash
npm test
```

**Coverage Targets**:
- Components: 70%+
- API clients: 80%+

---

### Integration Testing

**Test Scenarios**:

1. **End-to-End Registration Flow**
   ```
   1. POST /api/auth/register
   2. Verify email sent
   3. POST /api/auth/verify-email
   4. POST /api/auth/login
   5. Assert JWT token received
   ```

2. **Code Upload & Transformation**
   ```
   1. Login as user
   2. POST /api/projects (create project)
   3. POST /api/projects/:id/code (upload Python file)
   4. GET /api/projects/:id (verify code stored)
   5. POST /api/projects/:id/transform
   6. Assert transformed code is valid Python
   ```

3. **Cost Estimation**
   ```
   1. Upload code with storage operations
   2. GET /api/projects/:id/deployment-options
   3. Assert AWS and GCP options returned
   4. Assert costs are reasonable
   5. Assert recommendation is provided
   ```

4. **Resource Provisioning (AWS)**
   ```
   1. Upload code
   2. Select AWS + us-east-1
   3. Provide valid AWS credentials
   4. POST /api/projects/:id/provision
   5. Assert S3 bucket created in user's account
   6. Assert Terraform state stored in S3
   7. Cleanup: Delete bucket
   ```

5. **Resource Provisioning (GCP)**
   ```
   1. Upload code
   2. Select GCP + us-central1
   3. Provide valid GCP service account
   4. POST /api/projects/:id/provision
   5. Assert GCS bucket created in user's project
   6. Assert Terraform state stored in GCS
   7. Cleanup: Delete bucket
   ```

---

### Manual Testing Checklist

**Pre-MVP Testing**:

- [ ] Can register new account
- [ ] Email verification works
- [ ] Can login and get JWT
- [ ] JWT persists across page refresh
- [ ] Can upload Python file
- [ ] Syntax validation works
- [ ] Can see cost comparison
- [ ] Can select provider and region
- [ ] Can input resource names
- [ ] Bucket name uniqueness checked
- [ ] Can input AWS credentials
- [ ] AWS credentials validated
- [ ] Can input GCP credentials
- [ ] GCP credentials validated
- [ ] Transformed code is correct (AWS)
- [ ] Transformed code is correct (GCP)
- [ ] Can provision S3 bucket
- [ ] Can provision GCS bucket
- [ ] State stored correctly
- [ ] Can view provisioning logs
- [ ] Error handling works
- [ ] Can logout

**MVP Success Criteria**:

✅ **Complete Flow (AWS)**:
1. Register → Login
2. Upload `simple_storage.py`
3. See AWS cost: ~$23/month
4. Select AWS + us-east-1
5. Provide AWS credentials
6. Transform code → boto3 output
7. Provision S3 bucket
8. Manually deploy code (follow instructions)
9. Verify: Code can upload to S3 bucket

✅ **Complete Flow (GCP)**:
1. Login (existing user)
2. Upload same `simple_storage.py`
3. See GCP cost: ~$20/month (cheaper)
4. Select GCP + us-central1
5. Provide GCP service account
6. Transform code → google-cloud-storage output
7. Provision GCS bucket
8. Manually deploy code
9. Verify: Code can upload to GCS bucket

✅ **Provider Switch**:
1. Have project deployed on AWS
2. Click "Switch Provider"
3. Restart from step 3
4. New deployment config created
5. Old resources NOT destroyed (user does manually)

---

## Implementation Plan

### Phase 1: Foundation (Week 1-2)

**Goals**:
- Setup infrastructure
- Database & auth working
- Basic UI scaffolding

**Tasks**:

**Backend**:
- [ ] Create `infrar-platform` repository
- [ ] Setup Go project structure
- [ ] Setup PostgreSQL database
- [ ] Implement user registration endpoint
- [ ] Implement login endpoint
- [ ] JWT authentication middleware
- [ ] Database migrations

**Frontend**:
- [ ] Create `infrar-web` repository
- [ ] Setup Next.js project
- [ ] Setup Tailwind CSS + shadcn/ui
- [ ] Create signup page
- [ ] Create login page
- [ ] Create dashboard layout
- [ ] Implement auth state management

**Deliverable**: User can register and login

---

### Phase 2: Code Upload & Analysis (Week 3)

**Goals**:
- Upload Python files
- Validate syntax
- Detect capabilities

**Tasks**:

**Backend**:
- [ ] Implement project creation endpoint
- [ ] Implement code upload endpoint
- [ ] Python syntax validator
- [ ] Capability detector (detect `infrar.storage`)
- [ ] Store code in database/S3

**Frontend**:
- [ ] Create project creation page
- [ ] File upload component (drag & drop)
- [ ] Code editor (Monaco)
- [ ] Capability display

**Deliverable**: User can upload Python code and see detected capabilities

---

### Phase 3: Cost Intelligence (Week 4)

**Goals**:
- Pricing database setup
- Cost estimation working
- Provider comparison UI

**Tasks**:

**Backend**:
- [ ] Create pricing_data table
- [ ] Seed pricing data (AWS S3, GCP GCS)
- [ ] Cost estimation algorithm
- [ ] Deployment options endpoint
- [ ] Recommendation logic

**Frontend**:
- [ ] Cost comparison cards
- [ ] Resource breakdown table
- [ ] Provider selection UI
- [ ] Region selection

**Deliverable**: User sees cost comparison for AWS vs GCP

---

### Phase 4: Transformation (Week 5)

**Goals**:
- Code transformation working
- Parametrization implemented

**Tasks**:

**Backend**:
- [ ] Integrate infrar-engine as library
- [ ] Transformation endpoint
- [ ] Parametrization logic (replace bucket names)
- [ ] Store transformed code

**Frontend**:
- [ ] Before/after code view (split)
- [ ] Syntax highlighting
- [ ] Download transformed code

**Deliverable**: User sees transformed code for chosen provider

---

### Phase 5: Credentials & Provisioning (Week 6-7)

**Goals**:
- Secure credential storage
- OpenTofu execution working
- Resources created in user's account

**Tasks**:

**Backend**:
- [ ] Setup AWS Secrets Manager
- [ ] Credential storage endpoint
- [ ] Credential validation (AWS/GCP)
- [ ] OpenTofu executor service
- [ ] Generate Terraform configs dynamically
- [ ] Execute terraform init/plan/apply
- [ ] State storage in user's account
- [ ] Provisioning logs streaming

**Frontend**:
- [ ] Credentials input form
- [ ] Validation indicators
- [ ] Provisioning progress page
- [ ] Live logs display (WebSocket)
- [ ] Resource list

**Deliverable**: User provides credentials → S3/GCS bucket created in their account

---

### Phase 6: Testing & Polish (Week 8)

**Goals**:
- End-to-end testing
- Bug fixes
- Documentation

**Tasks**:
- [ ] Write integration tests
- [ ] Manual testing checklist
- [ ] Bug fixes
- [ ] Error handling improvements
- [ ] User documentation
- [ ] Demo video

**Deliverable**: MVP ready for demo

---

### Timeline Summary

| Week | Phase | Deliverable |
|------|-------|-------------|
| 1-2 | Foundation | User can register and login |
| 3 | Code Upload | User can upload Python code |
| 4 | Cost Intelligence | User sees cost comparison |
| 5 | Transformation | User sees transformed code |
| 6-7 | Provisioning | Resources created in user's account |
| 8 | Testing | MVP ready for demo |

**Total Time**: 8 weeks

---

## Risks & Mitigations

### Risk 1: Credential Security

**Risk**: User credentials leaked or stolen

**Mitigation**:
- ✅ Never store credentials in application database
- ✅ Use AWS Secrets Manager or Vault
- ✅ Encrypt credentials at rest and in transit
- ✅ Minimum permissions (read-only for validation)
- ✅ Audit logs for credential access
- ✅ Auto-rotate credentials (future)

---

### Risk 2: Terraform State Corruption

**Risk**: Terraform state file corrupted → resources orphaned

**Mitigation**:
- ✅ Enable S3 versioning for state files
- ✅ State locking (DynamoDB for AWS)
- ✅ Backup state files before apply
- ✅ Idempotent operations
- ✅ Manual recovery documentation

---

### Risk 3: Resource Naming Conflicts

**Risk**: User chooses bucket name that's already taken

**Mitigation**:
- ✅ Check bucket name availability before provisioning
- ✅ Suggest alternative names
- ✅ Fail gracefully with clear error message
- ✅ Allow retry with different name

---

### Risk 4: Provider API Rate Limits

**Risk**: Hit AWS/GCP API rate limits during provisioning

**Mitigation**:
- ✅ Exponential backoff
- ✅ Retry logic
- ✅ Queue system for high volume
- ✅ Rate limit monitoring

---

### Risk 5: Cost Estimate Accuracy

**Risk**: Cost estimates significantly off from actual costs

**Mitigation**:
- ✅ Update pricing data daily
- ✅ Add disclaimer: "Estimates only"
- ✅ Show pricing data age
- ✅ Link to provider pricing calculators
- ✅ Add buffer to estimates (10%)

---

### Risk 6: Transformation Errors

**Risk**: Transformed code doesn't work

**Mitigation**:
- ✅ Comprehensive test suite for transformations
- ✅ Validate transformed code syntax
- ✅ Integration tests with actual AWS/GCP
- ✅ Allow users to edit transformed code
- ✅ Provide error messages with fix suggestions

---

### Risk 7: Deployment Failures

**Risk**: Application deployment fails after resources created

**Mitigation**:
- ✅ Validate container builds locally first
- ✅ Provide clear deployment instructions (MVP)
- ✅ Health checks before marking success
- ✅ Rollback mechanism
- ✅ Deployment logs for debugging

---

## Next Steps

### Immediate Actions

1. **Create Repositories**
   ```bash
   # Create infrar-web
   mkdir infrar-web
   cd infrar-web
   npx create-next-app@latest .
   git init
   git add .
   git commit -m "Initial Next.js setup"

   # Create infrar-platform
   mkdir infrar-platform
   cd infrar-platform
   go mod init github.com/QodeSrl/infrar-platform
   git init
   ```

2. **Setup Infrastructure**
   - Provision PostgreSQL (AWS RDS or local Docker)
   - Setup AWS Secrets Manager
   - Create development environment

3. **Start Building**
   - Begin with Phase 1 (Foundation)
   - Focus on auth first
   - Get basic UI working

---

## Success Metrics

### MVP Demo Success

✅ **Technical Success**:
- User can register and login
- User can upload Python code
- User sees cost comparison
- User can transform code
- Resources created in user's cloud account
- Transformed code works with created resources

✅ **Business Success**:
- End-to-end flow takes < 10 minutes
- Cost estimates within 20% of actual
- Zero credential leaks
- 95% uptime during demo

---

## Conclusion

This MVP represents a complete end-to-end workflow for Infrar, demonstrating:
1. User experience (web UI)
2. Code transformation (core value prop)
3. Cost intelligence (competitive advantage)
4. Multi-cloud deployment (AWS + GCP)

**Estimated Effort**: 8 weeks full-time development

**Team Recommendation**:
- 1 Frontend Engineer (infrar-web)
- 1 Backend Engineer (infrar-platform)
- 1 DevOps Engineer (OpenTofu, deployment)

**MVP will prove**:
- Transformation engine works
- Multi-cloud deployment is real
- Cost intelligence provides value
- Complete workflow is achievable

Ready to build! 🚀
