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

Hetzner Cloud: infrar-production
└── Falkenstein (Germany) or Helsinki (Finland)
    │
    ├── Primary VPS (CPX31 or CCX23)
    │   ├── CPU: 4 vCPU (AMD EPYC)
    │   ├── RAM: 8 GB
    │   ├── Storage: 160 GB NVMe SSD
    │   ├── Network: 20 Gbit/s
    │   ├── OS: Ubuntu 22.04 LTS
    │   │
    │   ├── Docker Containers:
    │   │   ├── infrar-platform (Go API)
    │   │   │   ├── Port: 8080
    │   │   │   ├── Replicas: 2 (Docker Swarm)
    │   │   │   └── Auto-restart: always
    │   │   │
    │   │   ├── PostgreSQL 15
    │   │   │   ├── Port: 5432 (localhost only)
    │   │   │   ├── Data: /var/lib/postgresql
    │   │   │   └── Backups: Daily to Hetzner Backup
    │   │   │
    │   │   ├── Redis 7
    │   │   │   ├── Port: 6379 (localhost only)
    │   │   │   └── Maxmemory: 2GB
    │   │   │
    │   │   ├── Nginx (Reverse Proxy)
    │   │   │   ├── Port: 80, 443
    │   │   │   ├── SSL: Let's Encrypt (auto-renew)
    │   │   │   ├── Proxy: → infrar-platform:8080
    │   │   │   └── Rate limiting: enabled
    │   │   │
    │   │   ├── Prometheus (Monitoring)
    │   │   │   ├── Port: 9090 (localhost only)
    │   │   │   └── Retention: 15 days
    │   │   │
    │   │   └── Grafana (Dashboards)
    │   │       ├── Port: 3000 (localhost only)
    │   │       └── Dashboards: API metrics, DB stats
    │   │
    │   └── Firewall Rules:
    │       ├── Inbound: 22 (SSH), 80 (HTTP), 443 (HTTPS)
    │       └── Outbound: All allowed
    │
    ├── Hetzner Object Storage (S3-compatible)
    │   ├── Bucket: infrar-uploads
    │   ├── Bucket: infrar-artifacts
    │   ├── Bucket: infrar-logs
    │   ├── Location: Falkenstein (Germany)
    │   └── Pricing: €0.005/GB/month
    │
    ├── Hetzner Backup Service
    │   ├── VPS snapshots: Daily
    │   ├── Retention: 7 snapshots
    │   ├── Cost: 20% of VPS price
    │   └── Restore: < 1 hour
    │
    ├── Hetzner Floating IP
    │   ├── IP: api.infrar.io
    │   ├── Failover: Automatic
    │   └── Cost: €1.19/month
    │
    └── Hetzner Load Balancer (Optional, for scaling)
        ├── Type: LB11 (up to 20k connections)
        ├── Health Checks: /health
        ├── SSL Termination: Yes
        └── Cost: €5.39/month

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

### Monthly Cost Breakdown (Production - Hetzner + Cloudflare)

| Resource | Specification | Monthly Cost (€) | Monthly Cost ($) |
|----------|--------------|------------------|------------------|
| **Frontend** |
| Cloudflare Pages | Free tier | **€0** | **$0** ✅ |
| **Compute & Services** |
| Hetzner VPS CPX31 | 4 vCPU, 8GB RAM, 160GB NVMe | €13.90 | ~$15 |
| Hetzner Backup | Daily snapshots, 7-day retention | €2.78 | ~$3 |
| Hetzner Floating IP | Static IP for api.infrar.io | €1.19 | ~$1.30 |
| **Storage** |
| Hetzner Object Storage | 100GB | €0.50 | ~$0.55 |
| **Optional (Scaling)** |
| Hetzner Load Balancer | LB11 (if needed later) | €5.39 | ~$6 |
| **Domain & DNS** |
| Cloudflare DNS | Free | €0 | $0 |
| **Monitoring** |
| Self-hosted (Prometheus + Grafana) | Included in VPS | €0 | $0 |
| **Total (MVP)** | | **€18.37** | **~$20/month** ✅ |
| **Total (with Load Balancer)** | | **€23.76** | **~$26/month** |

**Cost Comparison**:
- **Hetzner**: €18-24/month (~$20-26)
- **AWS EU**: $540-850/month
- **Savings**: ~$520-830/month 🎉

**Notes**:
- 💰 **Extremely cost-effective**: 95%+ savings vs AWS
- 🇪🇺 **EU-based**: Servers in Germany/Finland (GDPR-compliant)
- 📈 **Scalable**: Can upgrade VPS or add load balancer
- 🔒 **Secure**: Isolated environment, firewall, backups
- ⚡ **Fast**: NVMe SSDs, 20 Gbit/s network

---

### Staging Environment Cost

| Resource | Monthly Cost |
|----------|--------------|
| Hetzner VPS CX21 | €5.83 (~$6.40) |
| Hetzner Backup | €1.17 (~$1.30) |
| Cloudflare Pages | $0 (Free) |
| **Total** | **~$7.70/month** ✅ |

**Savings vs AWS Staging**: ~$76/month

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

### **Cloudflare Pages + Hetzner Cloud** (Option 6)

**Rationale**:
1. ✅ **Cloudflare Pages**: Free, fast, global CDN for Next.js
2. ✅ **Hetzner Cloud**: EU-based, extremely cost-effective (~€50-100/month)
3. ✅ **EU Data Residency**: GDPR-compliant, servers in Germany/Finland
4. ✅ **Predictable Costs**: Fixed monthly pricing, no surprises
5. ✅ **High Performance**: NVMe SSDs, AMD EPYC CPUs, 20 Gbit/s network
6. ✅ **Simple Setup**: Easier than AWS, less complexity

**Resource Summary**:
```
Frontend: Cloudflare Pages (infrar.io) - Free
Backend API: Hetzner Cloud VPS (api.infrar.io)
Database: PostgreSQL on Hetzner VPS
Caching: Redis on same VPS
Storage: Hetzner Object Storage (S3-compatible)
Secrets: HashiCorp Vault on VPS or env variables
Monitoring: Prometheus + Grafana on VPS
Backups: Hetzner Backup Service
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

## Summary: Cloudflare Pages + Hetzner Cloud

### ✅ Final Architecture

**Frontend**: Cloudflare Pages (Free)
- Global CDN: 275+ cities
- Unlimited bandwidth
- Automatic HTTPS
- Git-based deployments
- **Cost**: €0/month ($0)

**Backend**: Hetzner Cloud (Germany/Finland)
- VPS CPX31: 4 vCPU, 8GB RAM
- PostgreSQL 15 (Docker)
- Redis 7 (Docker)
- Nginx reverse proxy
- Prometheus + Grafana
- Hetzner Object Storage (S3-compatible)
- Daily backups
- **Cost**: ~€18/month (~$20)

**Total Cost**: ~€18/month (~$20) ✅

### 💰 Cost Comparison

| Solution | Monthly Cost | Savings |
|----------|-------------|---------|
| **Hetzner + Cloudflare** | **~$20** | - |
| AWS + Vercel | $620 | **$600 saved** |
| AWS + Cloudflare | $540 | **$520 saved** |

**Hetzner saves 97% vs AWS!** 🎉

### 🇪🇺 EU Benefits

1. ✅ **GDPR Compliance**: Data stored in EU (Germany/Finland)
2. ✅ **Data Residency**: No data leaves EU region
3. ✅ **Low Latency**: Excellent for European users
4. ✅ **Regulatory Compliance**: Meets EU requirements
5. ✅ **Simple Setup**: No complex AWS configuration
6. ✅ **Predictable Costs**: Fixed monthly pricing

### 🚀 Deployment Flow

```
Developer pushes to GitHub
  ↓
Cloudflare Pages: Auto-build frontend (FREE)
  ↓
Hetzner VPS: Build Docker image
  ↓
Docker Swarm: Deploy containers (Go API, PostgreSQL, Redis, Nginx)
  ↓
Live: infrar.io (Cloudflare) + api.infrar.io (Hetzner)
```

### 📊 Hetzner VPS Options

| Plan | vCPU | RAM | Storage | Network | Price/month |
|------|------|-----|---------|---------|-------------|
| **CX21** (Staging) | 2 | 4GB | 40GB | 20 Gbit/s | €5.83 (~$6.40) |
| **CPX31** (Production) | 4 | 8GB | 160GB | 20 Gbit/s | €13.90 (~$15) ✅ |
| **CCX23** (Alternative) | 4 | 16GB | 160GB | 20 Gbit/s | €27.90 (~$31) |
| **CPX41** (Growth) | 8 | 16GB | 240GB | 20 Gbit/s | €27.90 (~$31) |

**For MVP**: CPX31 is perfect (4 vCPU, 8GB RAM, €13.90/month)

### 🔧 Technology Stack on Hetzner

```yaml
Operating System: Ubuntu 22.04 LTS
Container Runtime: Docker + Docker Compose
Orchestration: Docker Swarm (simple) or Kubernetes (later)

Containers:
  - infrar-platform (Go API)
  - PostgreSQL 15
  - Redis 7
  - Nginx (reverse proxy + SSL)
  - Prometheus (monitoring)
  - Grafana (dashboards)

Storage:
  - VPS NVMe: Code, databases, logs
  - Hetzner Object Storage: Uploads, artifacts (S3-compatible)

Backups:
  - Hetzner Backup: Daily VPS snapshots (7-day retention)
  - PostgreSQL dumps: Daily to Object Storage

Security:
  - UFW Firewall (ports 22, 80, 443 only)
  - Fail2ban (brute force protection)
  - Let's Encrypt SSL (auto-renewal)
  - SSH key authentication only
```

---

**Ready to proceed with repository creation!** 🎯
