# MVP Phase 1: Prove Transformation

**Duration**: 8 weeks
**Goal**: Demonstrate that compile-time code transformation works with real deployments and migrations

## Objectives

1. Build working transformation engine (infrar SDK → provider SDK)
2. Implement first plugin (infrar-storage) for AWS and GCP
3. Create deployment pipeline (GitHub → Transform → Deploy)
4. Prove migration works (AWS → GCP)
5. Build minimal web UI for management

## Scope

### In Scope

✅ **Language Support**:
- Python only

✅ **Providers**:
- AWS (ECS Fargate, S3, ECR)
- GCP (Cloud Run, Cloud Storage, Artifact Registry)

✅ **Capabilities**:
- `web_application`: Deploy containerized web apps
- Operations via `infrar-storage` plugin

✅ **Features**:
- Code transformation (AST-based)
- GitHub repository integration
- Docker image build
- Infrastructure provisioning (OpenTofu)
- Deployment to customer cloud accounts
- Provider migration (AWS ↔ GCP)
- Basic web UI
- Cost comparison (static pricing data)

### Out of Scope

❌ Azure support (Phase 2)
❌ Node.js support (Phase 2)
❌ Database plugin (Phase 2)
❌ Messaging plugin (Phase 2)
❌ Discovery of existing infrastructure (Phase 3)
❌ Dynamic pricing API (Phase 3)
❌ Advanced monitoring (Phase 3)
❌ Multi-container apps (Phase 2)
❌ Custom domains (Phase 2)

## Architecture

```
┌──────────────────┐
│  User's GitHub   │
│  (Python + infrar│
│   SDK code)      │
└────────┬─────────┘
         │
         ↓ Push webhook
┌────────────────────────────────┐
│    Infrar Platform (Go)        │
│                                │
│  ┌──────────────────────────┐  │
│  │  Transformation Engine   │  │
│  │  (Python AST parser)     │  │
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  Build Pipeline          │  │
│  │  (Docker build)          │  │
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  Deploy Engine           │  │
│  │  (OpenTofu)              │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
         │
         ↓
┌────────────────────────────────┐
│  AWS or GCP                    │
│  (Customer account)            │
│                                │
│  - Container running           │
│  - Transformed code (boto3/    │
│    google.cloud.storage)       │
└────────────────────────────────┘
```

## Deliverables

### 1. infrar-storage Plugin

**SDK** (`infrar-storage` Python package):
```python
# sdk/operations.py
def upload(bucket, source, destination):
    """Upload file to object storage."""
    pass

def download(bucket, source, destination):
    """Download file from object storage."""
    pass

def list_objects(bucket, prefix=''):
    """List objects in bucket."""
    pass

def delete(bucket, path):
    """Delete object from bucket."""
    pass
```

**Transformation Rules**:
- AWS: `infrar.storage.upload()` → `boto3.client('s3').upload_file()`
- GCP: `infrar.storage.upload()` → `storage.Client().bucket().blob().upload_from_filename()`

**OpenTofu Modules**:
- `aws/s3_bucket`: Create S3 bucket with proper configuration
- `gcp/cloud_storage_bucket`: Create GCS bucket

**Cost Models**:
- Storage: per GB per month
- Operations: per 1000 requests
- Data transfer: per GB egress

### 2. Transformation Engine

**Components**:
```
transformation_engine/
├── parser/
│   └── python_parser.py         # AST parsing
├── detector/
│   └── infrar_call_detector.py  # Find infrar SDK calls
├── transformer/
│   ├── transformation_rules.py  # Rule definitions
│   └── code_generator.py        # Generate provider code
└── validator/
    └── code_validator.py        # Validate generated code
```

**Input**: Python source code with `infrar` imports
**Output**: Python source code with provider SDKs (`boto3`, `google-cloud-storage`)

**Test Coverage**: 90%+ for transformation logic

### 3. Backend API (Go)

**Endpoints**:
```
POST   /api/projects                 # Create project
GET    /api/projects/:id             # Get project
PUT    /api/projects/:id             # Update project
DELETE /api/projects/:id             # Delete project

POST   /api/projects/:id/deploy      # Trigger deployment
GET    /api/deployments/:id          # Get deployment status
GET    /api/deployments/:id/logs     # Get deployment logs

POST   /api/projects/:id/migrate     # Trigger migration
GET    /api/migrations/:id           # Get migration status

POST   /api/credentials              # Add provider credentials
GET    /api/credentials              # List credentials
DELETE /api/credentials/:id          # Delete credentials

GET    /api/cost-comparison          # Get cost comparison
```

**Database Schema**:
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP
);

CREATE TABLE projects (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    repo_url VARCHAR(500) NOT NULL,
    dockerfile_path VARCHAR(255) DEFAULT './Dockerfile',
    created_at TIMESTAMP
);

CREATE TABLE deployments (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    provider VARCHAR(50) NOT NULL,  -- 'aws' or 'gcp'
    region VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,    -- 'pending', 'building', 'deploying', 'success', 'failed'
    endpoint_url VARCHAR(500),
    created_at TIMESTAMP,
    completed_at TIMESTAMP
);

CREATE TABLE provider_credentials (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    provider VARCHAR(50) NOT NULL,
    encrypted_credentials TEXT NOT NULL,
    created_at TIMESTAMP
);
```

### 4. Frontend (React)

**Pages**:

1. **Dashboard** (`/dashboard`):
   - List of projects
   - Quick stats (deployments, costs)
   - Create new project button

2. **Project Detail** (`/projects/:id`):
   - Project info (name, repo, current deployment)
   - Deployment history
   - Actions: Deploy, Migrate, Settings

3. **New Project Wizard** (`/projects/new`):
   - Step 1: GitHub repo URL
   - Step 2: Provider selection (AWS/GCP)
   - Step 3: Region selection
   - Step 4: Resource configuration (CPU, memory)
   - Step 5: Cost comparison
   - Step 6: Provider credentials
   - Step 7: Deploy

4. **Deployment View** (`/deployments/:id`):
   - Status (pending/building/deploying/success/failed)
   - Logs (live streaming)
   - Deployment details
   - Endpoint URL (when successful)

5. **Migration Wizard** (`/projects/:id/migrate`):
   - Current provider & cost
   - Target provider & estimated cost
   - Savings calculation
   - Confirm migration
   - Migration progress

**Technology**:
- React 18
- React Router for navigation
- TanStack Query for data fetching
- Tailwind CSS for styling
- WebSocket for live logs

### 5. Deployment Pipeline

**Workflow**:

```
1. User pushes to GitHub
   └─> GitHub webhook triggers Infrar

2. Infrar clones repo
   └─> Reads Dockerfile and source code

3. Code Transformation
   ├─> Parse all .py files
   ├─> Detect infrar SDK calls
   ├─> Generate provider-specific code
   └─> Write transformed files

4. Docker Build
   ├─> Create temporary directory
   ├─> Copy transformed code
   ├─> Copy Dockerfile
   ├─> Update requirements.txt (boto3 or google-cloud-storage)
   ├─> Build Docker image
   └─> Tag: {project_id}:{deployment_id}

5. Push to Registry
   ├─> AWS: Push to ECR
   └─> GCP: Push to Artifact Registry

6. Generate OpenTofu Config
   ├─> Select modules (ECS or Cloud Run)
   ├─> Fill in variables (image, cpu, memory, etc.)
   └─> Write .tf files

7. Provision Infrastructure
   ├─> tofu init
   ├─> tofu plan
   └─> tofu apply

8. Deploy Application
   ├─> ECS: Update service with new task definition
   └─> Cloud Run: Deploy new revision

9. Health Check
   ├─> Wait for healthy status
   └─> Validate endpoint responds

10. Update Status
    └─> Set deployment status to 'success'
```

**Error Handling**:
- Transformation errors → fail early, show helpful message
- Build errors → show Docker build logs
- Deployment errors → show OpenTofu errors
- Rollback on failure (keep previous deployment)

### 6. OpenTofu Modules

**AWS Module** (`modules/aws/web_application`):
```hcl
# main.tf
resource "aws_ecs_cluster" "main" {
  name = var.cluster_name
}

resource "aws_ecs_task_definition" "app" {
  family                   = var.app_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory

  container_definitions = jsonencode([{
    name  = var.app_name
    image = var.container_image
    portMappings = [{
      containerPort = var.container_port
      protocol      = "tcp"
    }]
    environment = var.environment_variables
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.app.name
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])

  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn
}

resource "aws_ecs_service" "app" {
  name            = var.app_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.min_instances
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.public[*].id
    security_groups = [aws_security_group.app.id]
    assign_public_ip = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = var.app_name
    container_port   = var.container_port
  }
}

# ... more resources (VPC, ALB, security groups, etc.)
```

**GCP Module** (`modules/gcp/web_application`):
```hcl
# main.tf
resource "google_cloud_run_service" "app" {
  name     = var.app_name
  location = var.region

  template {
    spec {
      containers {
        image = var.container_image

        resources {
          limits = {
            cpu    = var.cpu
            memory = "${var.memory}Mi"
          }
        }

        ports {
          container_port = var.container_port
        }

        dynamic "env" {
          for_each = var.environment_variables
          content {
            name  = env.key
            value = env.value
          }
        }
      }

      container_concurrency = 80
      timeout_seconds       = 300
    }

    metadata {
      annotations = {
        "autoscaling.knative.dev/minScale" = var.min_instances
        "autoscaling.knative.dev/maxScale" = var.max_instances
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

resource "google_cloud_run_service_iam_member" "public" {
  service  = google_cloud_run_service.app.name
  location = google_cloud_run_service.app.location
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# ... more resources
```

## Implementation Plan

### Week 1-2: Foundation

**Tasks**:
- [ ] Set up project structure (Go backend, React frontend)
- [ ] PostgreSQL database setup
- [ ] Authentication (JWT-based)
- [ ] Basic API endpoints (projects CRUD)
- [ ] Frontend scaffolding (React + routing)

**Deliverables**:
- Can create/list/view projects via UI
- Data persists in database

### Week 3-4: Transformation Engine

**Tasks**:
- [ ] Implement Python AST parser
- [ ] Build infrar call detector
- [ ] Create transformation rules for storage operations
- [ ] Implement code generator
- [ ] Write comprehensive tests
- [ ] Build `infrar-storage` Python package

**Deliverables**:
- `infrar.storage.upload()` transforms to `boto3` (AWS)
- `infrar.storage.upload()` transforms to `google.cloud.storage` (GCP)
- 90%+ test coverage

### Week 5-6: Build & Deploy Pipeline

**Tasks**:
- [ ] GitHub OAuth integration
- [ ] Webhook handling for push events
- [ ] Docker build pipeline
- [ ] Push to ECR/Artifact Registry
- [ ] OpenTofu integration (init, plan, apply)
- [ ] Create AWS ECS module
- [ ] Create GCP Cloud Run module
- [ ] Deployment status tracking

**Deliverables**:
- Push to GitHub triggers deployment
- App deployed on AWS or GCP
- Deployment logs visible in UI

### Week 7: Migration

**Tasks**:
- [ ] Migration workflow implementation
- [ ] Parallel deployment support
- [ ] Code re-transformation for new provider
- [ ] Migration status tracking
- [ ] Cleanup old infrastructure

**Deliverables**:
- Can migrate from AWS to GCP
- Can migrate from GCP to AWS
- Migration tracked in UI

### Week 8: Polish & Testing

**Tasks**:
- [ ] End-to-end testing
- [ ] Error handling improvements
- [ ] UI polish
- [ ] Cost comparison (static data)
- [ ] Documentation
- [ ] Demo preparation

**Deliverables**:
- Stable, working MVP
- Demo-ready
- User documentation

## Success Criteria

### Must Have

✅ Transform Python code with `infrar.storage` calls to AWS/GCP equivalents
✅ Deploy to AWS ECS Fargate successfully
✅ Deploy to GCP Cloud Run successfully
✅ Migrate from AWS to GCP without code changes
✅ Web UI for project management
✅ Deployment logs visible
✅ Basic cost comparison

### Nice to Have

- Deployment rollback
- Automated testing in CI/CD
- Performance metrics
- Advanced error handling

## Risks & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| AST transformation too complex | High | Medium | Start with simple API, expand gradually |
| OpenTofu state management issues | High | Medium | Use remote state from day 1, test thoroughly |
| GitHub webhook reliability | Medium | Low | Implement retry logic, manual trigger option |
| Customer credential security | Critical | Low | Encrypt at rest, audit all access, use Vault in production |
| Docker build performance | Medium | Medium | Use layer caching, consider build farm |

## Metrics

Track these metrics during MVP:
- Transformation success rate (target: 99%)
- Deployment success rate (target: 95%)
- Deployment time (target: < 10 minutes)
- Migration success rate (target: 90%)
- User satisfaction (NPS > 50)

## Next Phase

After Phase 1 completion, move to [Phase 2: Expand Capabilities](phase-2.md):
- Add Azure support
- Add Node.js transformation
- Add database plugin
- Add messaging plugin
- Multi-container support

## Demo Script

**Goal**: Show working deployment and migration in 10 minutes

1. **Setup** (pre-demo):
   - Sample Python app in GitHub
   - Uses `infrar.storage` to upload files
   - Dockerfile ready

2. **Create Project** (2 min):
   - Log into Infrar UI
   - Click "New Project"
   - Enter GitHub repo URL
   - Select AWS, us-east-1
   - Configure: 1 vCPU, 2GB RAM
   - Add AWS credentials
   - Click "Deploy"

3. **Watch Deployment** (3 min):
   - View logs in real-time
   - See transformation happening
   - See Docker build
   - See OpenTofu apply
   - Deployment succeeds
   - Click endpoint URL → app works!

4. **Test Application** (2 min):
   - Upload file via app
   - Verify file in S3 (show AWS console)

5. **Migrate to GCP** (3 min):
   - Click "Migrate"
   - Select GCP, us-central1
   - Show cost comparison (GCP cheaper)
   - Add GCP credentials
   - Click "Start Migration"
   - Watch parallel deployment
   - Migration completes
   - Click new endpoint → app works on GCP!
   - Upload file via app
   - Verify file in Cloud Storage (show GCP console)

6. **Show Code** (1 min):
   - Show original code (uses `infrar.storage`)
   - Show deployed AWS code (uses `boto3`)
   - Show deployed GCP code (uses `google.cloud.storage`)
   - **Same source code, different deployments!**

**Impact**: "Zero-overhead multi-cloud portability"
