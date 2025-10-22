# Infrar Infrastructure & Resource Placement Strategy

**Date**: October 2025
**Status**: Planning
**Purpose**: Define where all Infrar platform resources will be deployed and hosted

---

## Table of Contents

1. [Overview](#overview)
2. [Hosting Options Analysis](#hosting-options-analysis)
3. [Recommended Architecture](#recommended-architecture)
4. [Resource Placement by Environment](#resource-placement-by-environment)
5. [Cost Analysis](#cost-analysis)
6. [Security Considerations](#security-considerations)
7. [Scalability Plan](#scalability-plan)
8. [Decision Matrix](#decision-matrix)

---

## Overview

Infrar platform consists of multiple components that need to be hosted:

| Component | Type | Requirements |
|-----------|------|--------------|
| **infrar-web** | Static Frontend | CDN, HTTPS, High availability |
| **infrar-platform** | Backend API | Compute, Database access, Secrets access |
| **PostgreSQL** | Database | Persistent storage, Backups, High availability |
| **Pricing Data Sync** | Background Job | Scheduled execution, API access |
| **OpenTofu Executor** | Compute Service | Isolated execution, State management |
| **Container Builder** | Build Service | Docker daemon, Registry access |
| **File Storage** | Object Storage | Code files, artifacts, logs |
| **Secrets Storage** | Secrets Manager | Encrypted credentials |

---

## Hosting Options Analysis

### Option 1: Single Cloud Provider (AWS)

**Pros**:
- ✅ Simplified management
- ✅ Integrated services (RDS, Secrets Manager, ECS, S3)
- ✅ Lower latency between services
- ✅ Single bill
- ✅ IAM integration

**Cons**:
- ❌ Vendor lock-in (ironic for Infrar!)
- ❌ Single point of failure
- ❌ Regional limitations

**Resources**:
```
AWS Account (Infrar Corporate)
├── us-east-1 (Primary)
│   ├── VPC
│   │   ├── Public Subnets (ALB, NAT Gateway)
│   │   └── Private Subnets (ECS, RDS)
│   ├── ECS Fargate (infrar-platform API)
│   ├── RDS PostgreSQL (Database)
│   ├── ElastiCache Redis (Cache)
│   ├── S3 Buckets
│   │   ├── infrar-code-uploads
│   │   ├── infrar-artifacts
│   │   └── infrar-logs
│   ├── Secrets Manager (User credentials)
│   ├── CloudWatch (Logs & Metrics)
│   └── Application Load Balancer
└── CloudFront + S3 (Frontend - infrar-web)
```

**Estimated Cost**: $500-800/month

---

### Option 2: Multi-Cloud (AWS + Vercel)

**Pros**:
- ✅ Best tool for each job
- ✅ Vercel optimized for Next.js
- ✅ Global CDN for frontend
- ✅ Zero-config deployment
- ✅ Automatic HTTPS

**Cons**:
- ⚠️ Two vendors to manage
- ⚠️ CORS configuration needed
- ⚠️ Two bills

**Resources**:
```
Vercel (Frontend)
└── infrar-web
    ├── Global CDN
    ├── Automatic HTTPS
    ├── Preview deployments
    └── Edge functions (optional)

AWS Account (Backend)
└── us-east-1
    ├── ECS Fargate (infrar-platform)
    ├── RDS PostgreSQL
    ├── S3 (file storage)
    ├── Secrets Manager
    └── ALB
```

**Estimated Cost**: $600-900/month

---

### Option 3: Hybrid (AWS + Railway/Render)

**Pros**:
- ✅ Fast setup
- ✅ Managed services
- ✅ Good developer experience
- ✅ Auto-scaling

**Cons**:
- ⚠️ Higher cost at scale
- ⚠️ Less control
- ⚠️ Limited customization

**Resources**:
```
Railway/Render (Application)
├── infrar-web (static site)
├── infrar-platform (Go app)
└── PostgreSQL (managed)

AWS (Storage & Secrets)
├── S3 (file storage)
└── Secrets Manager
```

**Estimated Cost**: $700-1000/month

---

### Option 4: Self-Hosted (DigitalOcean/Hetzner)

**Pros**:
- ✅ Lower cost
- ✅ Full control
- ✅ Predictable pricing

**Cons**:
- ❌ More maintenance
- ❌ Need to manage backups
- ❌ Need to manage security
- ❌ Less scalable

**Not recommended for production**

---

## Recommended Architecture

### 🏆 **Option 2: Multi-Cloud (AWS + Vercel)**

**Rationale**:
1. **Vercel for Frontend**: Best-in-class Next.js hosting
2. **AWS for Backend**: Mature, reliable, integrated services
3. **Balanced**: Best tools for each job
4. **Cost-effective**: ~$600-900/month for MVP
5. **Scalable**: Can grow with us

---

## Resource Placement by Environment

### Development Environment

**Purpose**: Local development, testing, feature branches

```
Local Machine
├── Docker Compose
│   ├── infrar-web (localhost:3000)
│   ├── infrar-platform (localhost:8080)
│   ├── PostgreSQL (localhost:5432)
│   └── Redis (localhost:6379)
├── Local Files
│   ├── Uploaded code → /tmp/infrar/uploads
│   └── Artifacts → /tmp/infrar/artifacts
└── Mock Services
    ├── Mock AWS S3 (LocalStack)
    └── Mock Secrets Manager
```

**Setup**:
```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: infrar_dev
      POSTGRES_USER: infrar
      POSTGRES_PASSWORD: devpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  localstack:
    image: localstack/localstack
    environment:
      SERVICES: s3,secretsmanager
      AWS_DEFAULT_REGION: us-east-1
    ports:
      - "4566:4566"

volumes:
  postgres_data:
```

**Cost**: $0

---

### Staging Environment

**Purpose**: Pre-production testing, demo, QA

```
AWS Account: infrar-staging
└── us-east-1
    ├── VPC (10.1.0.0/16)
    │   ├── Public Subnets (2 AZs)
    │   └── Private Subnets (2 AZs)
    ├── ECS Fargate
    │   └── infrar-platform (1 task, 0.5 vCPU, 1GB RAM)
    ├── RDS PostgreSQL
    │   └── db.t3.micro (Single-AZ)
    ├── S3 Buckets
    │   ├── infrar-staging-uploads
    │   ├── infrar-staging-artifacts
    │   └── infrar-staging-logs
    ├── Secrets Manager
    └── ALB (Application Load Balancer)

Vercel: staging
└── infrar-web (staging.infrar.io)
    └── Connected to staging API
```

**Configuration**:
- **Database**: db.t3.micro (2 vCPU, 1GB RAM, 20GB storage)
- **Backend**: 1 ECS task (0.5 vCPU, 1GB RAM)
- **Frontend**: Vercel staging environment
- **Backups**: Daily snapshots, 7-day retention

**Cost**: ~$150-200/month

---

### Production Environment

**Purpose**: Live platform serving users

```
Cloudflare Pages: infrar-web (Frontend)
└── Global CDN (275+ edge locations)
    ├── Custom domain: infrar.io, www.infrar.io
    ├── Automatic HTTPS
    ├── Unlimited bandwidth (FREE)
    ├── Build: Next.js static export
    ├── Edge Functions: Optional (serverless)
    └── Environment Variables
        └── NEXT_PUBLIC_API_URL=https://api.infrar.io

AWS Account: infrar-production
└── eu-west-1 (Ireland - Primary EU Region)
    ├── VPC (10.0.0.0/16)
    │   ├── Public Subnets (3 AZs)
    │   │   ├── NAT Gateways
    │   │   └── Application Load Balancer
    │   └── Private Subnets (3 AZs)
    │       ├── ECS Tasks
    │       └── RDS Instances
    │
    ├── ECS Fargate Cluster
    │   ├── infrar-platform-api
    │   │   ├── Min: 2 tasks
    │   │   ├── Max: 10 tasks
    │   │   └── Task: 1 vCPU, 2GB RAM
    │   │
    │   ├── infrar-pricing-sync (Scheduled)
    │   │   └── Runs daily at 02:00 UTC
    │   │
    │   └── infrar-terraform-executor
    │       ├── Min: 0 tasks
    │       ├── Max: 5 tasks
    │       └── Task: 2 vCPU, 4GB RAM
    │
    ├── RDS PostgreSQL
    │   ├── Instance: db.t3.medium (Multi-AZ)
    │   ├── Storage: 100GB GP3
    │   ├── Backups: Daily, 30-day retention
    │   ├── Replica: Read replica in us-east-1
    │   └── Encryption: AES-256
    │
    ├── ElastiCache Redis
    │   ├── Node: cache.t3.micro
    │   └── Replicas: 1
    │
    ├── S3 Buckets
    │   ├── infrar-prod-uploads
    │   │   ├── Encryption: AES-256
    │   │   ├── Versioning: Enabled
    │   │   └── Lifecycle: Delete after 90 days
    │   │
    │   ├── infrar-prod-artifacts
    │   │   ├── Encryption: AES-256
    │   │   └── Lifecycle: Move to Glacier after 30 days
    │   │
    │   └── infrar-prod-logs
    │       ├── Encryption: AES-256
    │       └── Lifecycle: Delete after 30 days
    │
    ├── Secrets Manager
    │   ├── User AWS credentials
    │   ├── User GCP credentials
    │   ├── Database credentials
    │   └── API keys (Stripe, SendGrid, etc.)
    │
    ├── CloudWatch
    │   ├── Logs (all services)
    │   ├── Metrics (CPU, memory, requests)
    │   ├── Alarms (error rates, latency)
    │   └── Dashboards
    │
    ├── Application Load Balancer
    │   ├── Target Group: ECS tasks
    │   ├── Health Checks: /health
    │   ├── SSL/TLS: Certificate Manager
    │   └── WAF: Rate limiting, IP filtering
    │
    └── Route 53
        ├── api.infrar.io → ALB
        └── Health checks

```

**Configuration**:

**Frontend (Cloudflare Pages)**:
- Framework: Next.js 14+ with static export
- Build Command: `npm run build && npm run export`
- Output Directory: `out/`
- Custom Domain: infrar.io (via Cloudflare DNS)
- SSL/TLS: Automatic (Free)
- CDN: 275+ global edge locations
- Build Concurrency: Unlimited
- Bandwidth: Unlimited (Free)
- Preview Deployments: Automatic for PRs
- Environment Variables: Managed via dashboard
- Analytics: Cloudflare Web Analytics (Free)
- Functions: Cloudflare Pages Functions (optional, serverless)

**Cloudflare Pages Features**:
```yaml
Free Tier Includes:
  - Unlimited sites
  - Unlimited requests
  - Unlimited bandwidth
  - 500 builds/month
  - 1 concurrent build
  - 20 minutes build time per deployment

Pro Tier ($20/month):
  - 5,000 builds/month
  - 5 concurrent builds
  - 30 minutes build time
  - Advanced build configuration
```

**For MVP**: Free tier is sufficient

**Backend (ECS Fargate - eu-west-1)**:
- Task Definition: 1 vCPU, 2GB RAM
- Auto-scaling: 2-10 tasks
- Health Check: `/health` endpoint every 30s
- Rolling updates: 50% at a time

**Database (RDS PostgreSQL)**:
- Instance: db.t3.medium (2 vCPU, 4GB RAM)
- Storage: 100GB GP3 (3000 IOPS)
- Multi-AZ: Enabled (automatic failover)
- Backups: Daily automated, 30-day retention
- Read Replica: 1 replica for read scaling
- Connection Pooling: PgBouncer on ECS

**Caching (ElastiCache Redis)**:
- Node: cache.t3.micro (0.5GB memory)
- Replicas: 1 (for high availability)
- Use Cases: Session storage, API rate limiting

**File Storage (S3)**:
- Encryption: Server-side AES-256
- Versioning: Enabled for uploads
- Lifecycle Policies:
  - Uploads: Delete after 90 days
  - Artifacts: Move to Glacier after 30 days
  - Logs: Delete after 30 days

**Secrets (AWS Secrets Manager)**:
- Automatic rotation: Every 90 days
- Encryption: AWS KMS
- Access: ECS task role only

**Frontend (Vercel)**:
- Framework: Next.js 14+ (App Router)
- Deployment: Git push → auto-deploy
- CDN: Global edge network (200+ locations)
- SSL: Automatic (Let's Encrypt)
- Domains: infrar.io, www.infrar.io

**Cost**: ~$600-900/month (MVP scale)

---

## Detailed Resource Specifications

### Networking

**VPC Configuration (eu-west-1 - Ireland)**:
```
VPC: 10.0.0.0/16
├── Public Subnets (Internet access)
│   ├── eu-west-1a: 10.0.1.0/24
│   ├── eu-west-1b: 10.0.2.0/24
│   └── eu-west-1c: 10.0.3.0/24
└── Private Subnets (NAT Gateway access)
    ├── eu-west-1a: 10.0.11.0/24
    ├── eu-west-1b: 10.0.12.0/24
    └── eu-west-1c: 10.0.13.0/24

Internet Gateway: Attached to VPC
NAT Gateways: 1 per public subnet (high availability)
```

**Security Groups**:
```
ALB Security Group
├── Inbound: 443 (HTTPS) from 0.0.0.0/0
└── Outbound: 8080 to ECS Security Group

ECS Security Group
├── Inbound: 8080 from ALB Security Group
└── Outbound: 443 (HTTPS), 5432 (PostgreSQL)

RDS Security Group
├── Inbound: 5432 from ECS Security Group
└── Outbound: None

Redis Security Group
├── Inbound: 6379 from ECS Security Group
└── Outbound: None
```

---

### Compute Resources

**ECS Task Definition (infrar-platform)**:
```json
{
  "family": "infrar-platform",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "infrar-platform",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/infrar-platform:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        },
        {
          "name": "DATABASE_HOST",
          "value": "infrar-prod.xxx.us-east-1.rds.amazonaws.com"
        },
        {
          "name": "REDIS_URL",
          "value": "redis://infrar-redis.xxx.cache.amazonaws.com:6379"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:xxx:secret:infrar/db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/infrar-platform",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::xxx:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::xxx:role/infraPlatformTaskRole"
}
```

**Auto-Scaling Policy**:
```json
{
  "targetTrackingScalingPolicyConfiguration": {
    "targetValue": 70.0,
    "predefinedMetricSpecification": {
      "predefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "scaleOutCooldown": 60,
    "scaleInCooldown": 300
  }
}
```

---

### Database

**RDS PostgreSQL Configuration**:
```
Instance Class: db.t3.medium
  - vCPUs: 2
  - RAM: 4 GB
  - Network: Up to 5 Gbps

Storage:
  - Type: GP3 (General Purpose SSD)
  - Size: 100 GB
  - IOPS: 3000
  - Throughput: 125 MB/s
  - Auto-scaling: Enabled (max 500 GB)

High Availability:
  - Multi-AZ: Enabled
  - Automatic failover: < 60 seconds
  - Backup: Daily, 30-day retention

Security:
  - Encryption at rest: AES-256
  - Encryption in transit: SSL/TLS
  - IAM authentication: Enabled

Monitoring:
  - Enhanced monitoring: 60-second granularity
  - Performance Insights: Enabled

Maintenance:
  - Maintenance window: Sunday 03:00-04:00 UTC
  - Auto minor version upgrade: Enabled
```

**Connection Pooling (PgBouncer)**:
```
Deployed as ECS sidecar container
- Pool mode: Transaction
- Max connections: 100
- Default pool size: 25
- Reserve pool size: 5
```

---

### Storage

**S3 Bucket Configurations**:

**infrar-prod-uploads** (User code uploads):
```json
{
  "Versioning": "Enabled",
  "Encryption": {
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  },
  "LifecycleConfiguration": {
    "Rules": [
      {
        "Id": "DeleteOldUploads",
        "Status": "Enabled",
        "ExpirationInDays": 90
      }
    ]
  },
  "PublicAccessBlock": {
    "BlockPublicAcls": true,
    "BlockPublicPolicy": true,
    "IgnorePublicAcls": true,
    "RestrictPublicBuckets": true
  }
}
```

**infrar-prod-artifacts** (Transformed code, Docker images metadata):
```json
{
  "Versioning": "Disabled",
  "Encryption": {
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  },
  "LifecycleConfiguration": {
    "Rules": [
      {
        "Id": "MoveToGlacier",
        "Status": "Enabled",
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "GLACIER"
          }
        ]
      }
    ]
  }
}
```

---

### Secrets Management

**AWS Secrets Manager Structure**:
```
/infrar/production/
├── database
│   ├── master-password
│   └── application-password
├── users/
│   ├── {project_id}/aws-credentials
│   └── {project_id}/gcp-credentials
├── api-keys/
│   ├── stripe-secret-key
│   ├── sendgrid-api-key
│   └── github-oauth-secret
└── jwt/
    ├── signing-key
    └── refresh-key
```

**Access Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::xxx:role/infraPlatformTaskRole"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:xxx:secret:/infrar/production/*"
    }
  ]
}
```

**Rotation Policy**:
- Database passwords: Every 90 days
- API keys: Manual rotation
- User credentials: No automatic rotation (user-managed)

---

## Cost Analysis

### Monthly Cost Breakdown (Production - EU Region)

| Resource | Specification | Monthly Cost (EU) |
|----------|--------------|-------------------|
| **Frontend** |
| Cloudflare Pages | Free tier | **$0** ✅ |
| **Compute** |
| ECS Fargate (API) | 2-10 tasks, 1vCPU, 2GB | $150-750 |
| ECS Fargate (Jobs) | On-demand | $20-50 |
| **Database** |
| RDS PostgreSQL | db.t3.medium, Multi-AZ (EU) | $150 |
| RDS Storage | 100GB GP3 (EU) | $13 |
| RDS Backups | 100GB | $11 |
| **Caching** |
| ElastiCache Redis | cache.t3.micro (EU) | $16 |
| **Storage** |
| S3 Standard (EU) | 100GB | $2.50 |
| S3 Glacier (EU) | 500GB (after 30 days) | $2.10 |
| **Networking** |
| Application Load Balancer | 1 ALB | $24 |
| NAT Gateway | 3 gateways (EU) | $105 |
| Data Transfer Out | 100GB | $9 |
| **Secrets** |
| Secrets Manager | 50 secrets | $20 |
| **Monitoring** |
| CloudWatch Logs | 10GB | $5 |
| CloudWatch Metrics | Custom metrics | $10 |
| **Container Registry** |
| ECR | 10GB storage | $1 |
| **Total (Low traffic)** | | **~$540/month** |
| **Total (Medium traffic)** | | **~$850/month** |
| **Total (High traffic)** | | **~$1,200/month** |

**Notes**:
- Low traffic: < 10 users, < 100 deployments/month
- Medium traffic: 50 users, 500 deployments/month
- High traffic: 200 users, 2000 deployments/month
- **Cloudflare Pages savings**: ~$20/month vs Vercel Pro
- EU pricing is ~7-10% higher than US regions
- GDPR-compliant data residency in EU

---

### Staging Environment Cost

| Resource | Monthly Cost |
|----------|--------------|
| ECS Fargate (1 task) | $30 |
| RDS db.t3.micro | $20 |
| S3 Storage | $2 |
| ALB | $22 |
| Secrets Manager | $5 |
| CloudWatch | $5 |
| Vercel (included in Pro) | $0 |
| **Total** | **~$84/month** |

---

### Development Environment Cost

| Resource | Cost |
|----------|------|
| Local (Docker) | $0 |
| Total | **$0/month** |

---

## Security Considerations

### Network Security

**Defense in Depth**:
```
Internet
  ↓
CloudFront / Vercel (Frontend)
  ↓
WAF (Web Application Firewall)
  ↓
Application Load Balancer (HTTPS only)
  ↓
Private Subnet (ECS Tasks)
  ↓
Private Subnet (RDS, Redis)
```

**WAF Rules**:
- Rate limiting: 2000 requests/5min per IP
- Geographic restrictions: Optional
- SQL injection protection
- XSS protection
- Known bad inputs blocking

---

### Data Security

**Encryption**:
- ✅ **At Rest**: All S3, RDS, Secrets Manager encrypted (AES-256)
- ✅ **In Transit**: TLS 1.2+ for all connections
- ✅ **Application Level**: User credentials encrypted before storage

**Access Control**:
- ✅ **IAM Roles**: Least privilege principle
- ✅ **MFA**: Required for production access
- ✅ **Audit Logs**: CloudTrail enabled for all API calls
- ✅ **Security Groups**: Whitelist only necessary ports

---

### Compliance

**Data Residency**:
- Default region: us-east-1 (USA)
- User data: Can be stored in user's chosen region (future)
- Backup retention: 30 days

**GDPR Considerations**:
- User data deletion: On request
- Data export: Available via API
- Privacy policy: Clear and accessible

---

## Scalability Plan

### Phase 1: MVP (0-100 users)

**Current Architecture** (as described above)
- 2-10 ECS tasks
- Single RDS instance (Multi-AZ)
- Single Redis node

**Capacity**:
- ✅ 100 concurrent users
- ✅ 1000 deployments/month
- ✅ < 100ms API response time

---

### Phase 2: Growth (100-1000 users)

**Scaling Changes**:
- ECS tasks: 5-50 (auto-scaling)
- RDS: Add read replicas (2-3)
- Redis: Add replicas (cluster mode)
- S3: Enable cross-region replication
- Add CDN for API responses (CloudFront)

**Capacity**:
- ✅ 1000 concurrent users
- ✅ 10,000 deployments/month
- ✅ < 150ms API response time

**Additional Cost**: +$1,000-2,000/month

---

### Phase 3: Scale (1000+ users)

**Architecture Changes**:
- Multi-region deployment (us-east-1, eu-west-1, ap-southeast-1)
- Database sharding by region
- Kubernetes (EKS) for better orchestration
- Separate compute clusters for different workloads
- API gateway with rate limiting per user tier

**Capacity**:
- ✅ 10,000+ concurrent users
- ✅ 100,000+ deployments/month
- ✅ < 200ms API response time (global)

**Additional Cost**: +$5,000-10,000/month

---

## Decision Matrix

### Choosing Between Options

| Criteria | AWS Only | AWS + Vercel | AWS + Railway | Self-Hosted |
|----------|----------|--------------|---------------|-------------|
| **Setup Time** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Cost (MVP)** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalability** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Maintenance** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Developer Experience** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Control** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Security** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## 🏆 Final Recommendation

### **Cloudflare Pages + AWS EU** (Option 5)

**Rationale**:
1. ✅ **Cloudflare Pages**: Free, fast, global CDN for Next.js
2. ✅ **EU Data Residency**: GDPR-compliant, backend in EU region
3. ✅ **Cost-Effective**: Cloudflare Pages is free (unlimited bandwidth)
4. ✅ **Scalable**: Both platforms scale well
5. ✅ **Global Performance**: Cloudflare's CDN (275+ cities) + AWS EU
6. ✅ **Maintainable**: Managed services reduce ops burden

**Resource Summary**:
```
Frontend: Cloudflare Pages (infrar.io) - Free
Backend API: AWS ECS Fargate eu-west-1 (api.infrar.io)
Database: AWS RDS PostgreSQL eu-west-1
Caching: AWS ElastiCache Redis eu-west-1
Storage: AWS S3 eu-west-1
Secrets: AWS Secrets Manager eu-west-1
Monitoring: AWS CloudWatch eu-west-1
```

---

## Implementation Steps

### Step 1: Setup AWS Account (EU Region)

```bash
# Create AWS account (if not exists)
# Enable MFA
# Create IAM user for Terraform
# Configure billing alerts
# Set primary region to eu-west-1 (Ireland)
```

### Step 2: Setup Cloudflare Account

```bash
# Sign up at cloudflare.com (free account)
# Add domain: infrar.io
# Configure DNS settings
# Create Cloudflare Pages project
# Connect to GitHub repository
```

**Cloudflare Pages Setup**:
1. Go to https://pages.cloudflare.com
2. Connect GitHub account
3. Select `infrar-web` repository
4. Configure build settings:
   - Build command: `npm run build`
   - Build output directory: `out`
   - Root directory: `/`
5. Add custom domain: `infrar.io`
6. Deploy!

### Step 3: Infrastructure as Code

```bash
# Create infrar-infrastructure repository
mkdir infrar-infrastructure
cd infrar-infrastructure

# Initialize Terraform
terraform init

# Create resources
terraform plan
terraform apply
```

### Step 4: Deploy Services

```bash
# Backend: Build and push Docker image to ECR (EU)
cd infrar-platform
docker build -t infrar-platform .
docker push 123456789.dkr.ecr.eu-west-1.amazonaws.com/infrar-platform

# Frontend: Git push triggers Cloudflare Pages deployment
cd infrar-web
git push origin main
# Cloudflare Pages automatically builds and deploys!
```

---

## Next Steps

1. ✅ **Review this document**
2. ✅ **Approve resource placement strategy**
3. ⏭️ **Create AWS account (production)**
4. ⏭️ **Create Vercel account**
5. ⏭️ **Create `infrar-infrastructure` repository**
6. ⏭️ **Write Terraform code for AWS resources**
7. ⏭️ **Setup CI/CD pipelines**

---

## Summary: Cloudflare Pages + AWS EU

### ✅ Final Architecture

**Frontend**: Cloudflare Pages (Free)
- Global CDN: 275+ cities
- Unlimited bandwidth
- Automatic HTTPS
- Git-based deployments
- **Cost**: $0/month

**Backend**: AWS eu-west-1 (Ireland)
- ECS Fargate for API
- RDS PostgreSQL (Multi-AZ)
- ElastiCache Redis
- S3 for storage
- Secrets Manager
- **Cost**: ~$540-850/month

**Total Cost**: ~$540-850/month (saves $20/month vs Vercel)

### 🇪🇺 EU Benefits

1. ✅ **GDPR Compliance**: Data stored in EU
2. ✅ **Data Residency**: No data leaves EU region
3. ✅ **Low Latency**: For European users
4. ✅ **Regulatory Compliance**: Meets EU requirements

### 🚀 Deployment Flow

```
Developer pushes to GitHub
  ↓
Cloudflare Pages: Auto-build frontend (FREE)
  ↓
AWS ECR (eu-west-1): Build backend Docker image
  ↓
AWS ECS (eu-west-1): Deploy to Fargate
  ↓
Live: infrar.io (Cloudflare) + api.infrar.io (AWS EU)
```

---

**Ready to proceed with repository creation!** 🎯
