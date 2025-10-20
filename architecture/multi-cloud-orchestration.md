# Multi-Cloud Orchestration

## Overview

Multi-cloud orchestration enables running different capabilities on different cloud providers simultaneously. This is a key differentiator for Infrar - not just migration between providers, but **partial migration** and **multi-cloud by design**.

## Use Case

A company discovers they can optimize costs by running different workloads on different providers:

```
Before (All on AWS): $1,000/month
├─ Data Intelligence: $500/month
├─ Web Application: $320/month
└─ Background Workers: $180/month

After (Multi-Cloud): $725/month (27.5% savings)
├─ Data Intelligence on Azure: $340/month (-32%)
├─ Web Application on AWS: $320/month (unchanged)
└─ Background Workers on GCP: $65/month (-64%)
```

**Challenge**: How do these services communicate across clouds?

## Architecture Patterns

### Pattern 1: Public Internet Communication

**Simplest approach**: Services communicate over public HTTPS.

```
┌────────────────────────────┐
│       AWS (us-east-1)      │
│                            │
│  ┌──────────────────────┐  │
│  │  Web Application     │  │
│  │  (ECS Fargate)       │  │
│  │                      │  │
│  │  app.backend.com     │  │
│  └──────────┬───────────┘  │
└─────────────┼──────────────┘
              │ HTTPS
              │ (public internet)
              ↓
┌────────────────────────────┐
│      Azure (eastus)        │
│                            │
│  ┌──────────────────────┐  │
│  │  Data Service        │  │
│  │  (Container Apps)    │  │
│  │                      │  │
│  │  data.api.com        │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**Pros**:
- Simple to implement
- No special networking configuration
- Works immediately

**Cons**:
- Data egress costs ($0.09-$0.12/GB)
- Latency (public internet routing)
- Less secure (requires public endpoints)

**When to use**: MVP, low data transfer, non-sensitive data

### Pattern 2: VPN Connection

**More secure**: Establish VPN tunnel between clouds.

```
┌────────────────────────────┐
│       AWS (us-east-1)      │
│                            │
│  ┌──────────────────────┐  │
│  │  VPC: 10.0.0.0/16    │  │
│  │                      │  │
│  │  ┌────────────────┐  │  │
│  │  │ Web App        │  │  │
│  │  │ 10.0.1.10      │  │  │
│  │  └───────┬────────┘  │  │
│  └──────────┼───────────┘  │
│             │              │
│  ┌──────────▼───────────┐  │
│  │  VPN Gateway         │  │
│  │  (AWS VPN)           │  │
│  └──────────┬───────────┘  │
└─────────────┼──────────────┘
              │ Encrypted tunnel
              │ (IPsec VPN)
              ↓
┌─────────────▼──────────────┐
│      Azure (eastus)        │
│                            │
│  ┌──────────────────────┐  │
│  │  VPN Gateway         │  │
│  │  (Azure VPN)         │  │
│  └──────────┬───────────┘  │
│             │              │
│  ┌──────────▼───────────┐  │
│  │  VNet: 10.1.0.0/16   │  │
│  │                      │  │
│  │  ┌────────────────┐  │  │
│  │  │ Data Service   │  │  │
│  │  │ 10.1.1.10      │  │  │
│  │  └────────────────┘  │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**Pros**:
- Private communication
- More secure
- Reduced egress costs (still charged, but lower)

**Cons**:
- Complex setup
- VPN gateway costs (~$36/month per side)
- Still has some data egress charges
- Latency overhead (encryption/decryption)

**When to use**: Sensitive data, high security requirements

### Pattern 3: Private Peering

**Highest performance**: Direct private connection between clouds.

```
┌────────────────────────────┐
│       AWS (us-east-1)      │
│                            │
│  ┌──────────────────────┐  │
│  │  VPC: 10.0.0.0/16    │  │
│  │  Web App             │  │
│  └──────────┬───────────┘  │
│             │              │
│  ┌──────────▼───────────┐  │
│  │  AWS PrivateLink     │  │
│  └──────────┬───────────┘  │
└─────────────┼──────────────┘
              │ Private connection
              │ (Azure Private Link)
              ↓
┌─────────────▼──────────────┐
│      Azure (eastus)        │
│                            │
│  ┌──────────────────────┐  │
│  │  Azure Private Link  │  │
│  └──────────┬───────────┘  │
│             │              │
│  ┌──────────▼───────────┐  │
│  │  VNet: 10.1.0.0/16   │  │
│  │  Data Service        │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**Pros**:
- Highest performance
- Most secure
- No egress charges for some data types
- Low latency

**Cons**:
- Most complex setup
- Requires coordination with both providers
- Not available in all regions
- Higher monthly costs

**When to use**: Production, high-throughput, enterprise

## Infrar's Multi-Cloud Networking

### MVP Approach (Pattern 1)

For MVP, use public internet with HTTPS:

**Benefits**:
- Simple to implement
- No networking configuration needed
- Works immediately
- Good for proving concept

**Implementation**:

```python
# Web Application on AWS
import requests

# Call Azure data service
response = requests.post(
    'https://data-service.azure.infrar.io/query',
    json={'query': 'SELECT * FROM sales'},
    headers={'Authorization': f'Bearer {get_token()}'}
)
```

**Infrar handles**:
- DNS configuration (data-service.azure.infrar.io → Azure endpoint)
- Authentication tokens (via Infrar auth service)
- Request routing
- Cost tracking (egress charges)

### Production Approach (Pattern 2/3)

For production, Infrar can provision VPN or private peering:

```yaml
# project.yaml

multi_cloud:
  enabled: true
  networking_mode: vpn  # or 'public', 'private_peering'

  connections:
    - from: aws_us_east_1
      to: azure_eastus
      type: vpn
      bandwidth: 1gbps

    - from: aws_us_east_1
      to: gcp_us_central
      type: public  # Still public for low-traffic services
```

## Service Discovery

### Challenge

How does Web Application on AWS know where Data Service on Azure is?

### Solution: Infrar Service Registry

Infrar provides a service registry that tracks all deployed services:

```python
# infrar.registry module (included in transformed code)

import infrar.registry as registry

# Get endpoint for data service (regardless of provider)
data_service_url = registry.get_endpoint('data-intelligence')

# Makes request to correct provider
response = requests.post(
    f'{data_service_url}/query',
    json={'query': 'SELECT * FROM sales'},
    headers={'Authorization': registry.get_auth_token('data-intelligence')}
)
```

**Under the hood**:
```python
# Infrar registry resolves to actual endpoint
{
    'data-intelligence': {
        'url': 'https://data-service.azure.infrar.io',
        'provider': 'azure',
        'region': 'eastus',
        'internal_url': '10.1.1.10',  # If VPN enabled
        'auth_method': 'bearer_token'
    }
}
```

## Authentication & Authorization

### Cross-Cloud Identity

Each service needs to authenticate with services on other providers.

**Infrar Auth Service**:

```
┌────────────────────────────┐
│       AWS (us-east-1)      │
│                            │
│  ┌──────────────────────┐  │
│  │  Web Application     │  │
│  │                      │  │
│  │  1. Get token from   │  │
│  │     Infrar Auth      │  │
│  └──────────┬───────────┘  │
└─────────────┼──────────────┘
              │
              ↓ 2. Request token for 'data-intelligence'
┌─────────────────────────────────────┐
│      Infrar Auth Service            │
│      (Centralized)                  │
│                                     │
│  3. Issues JWT token:               │
│     - Subject: web-app              │
│     - Audience: data-intelligence   │
│     - Expiry: 1 hour                │
└─────────────┬───────────────────────┘
              │
              ↓ 4. Use token in request
┌─────────────▼──────────────┐
│      Azure (eastus)        │
│                            │
│  ┌──────────────────────┐  │
│  │  Data Service        │  │
│  │                      │  │
│  │  5. Validate token   │  │
│  │     with Infrar      │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**Implementation**:

```python
# Web Application (AWS)
def query_data():
    # Get token from Infrar auth service
    token = infrar.auth.get_token(
        service='data-intelligence',
        scopes=['read:data', 'write:query']
    )

    # Use token
    response = requests.post(
        infrar.registry.get_endpoint('data-intelligence') + '/query',
        headers={'Authorization': f'Bearer {token}'},
        json={'query': 'SELECT * FROM sales'}
    )
    return response.json()

# Data Service (Azure)
@app.route('/query', methods=['POST'])
def handle_query():
    # Validate token
    token = request.headers.get('Authorization').replace('Bearer ', '')

    try:
        claims = infrar.auth.validate_token(token)
        # Check permissions
        if 'write:query' not in claims['scopes']:
            return {'error': 'Insufficient permissions'}, 403

        # Execute query
        result = execute_query(request.json['query'])
        return {'result': result}

    except infrar.auth.InvalidToken:
        return {'error': 'Invalid token'}, 401
```

## Data Migration Between Clouds

### Scenario

User wants to migrate "data intelligence" capability from AWS to Azure, but keep web application on AWS.

**Before**:
```
AWS:
├─ Web Application (ECS)
└─ Data Intelligence (Athena + S3)
    └─ 5TB of data in S3
```

**After**:
```
AWS:
└─ Web Application (ECS)

Azure:
└─ Data Intelligence (Synapse + Blob)
    └─ 5TB of data in Blob Storage
```

### Migration Process

```
Phase 1: Parallel Deployment
┌────────────────────┐
│  AWS S3            │
│  (5TB data)        │  ────┐
│  ✅ Active         │      │
└────────────────────┘      │
                            │ Infrar copies data
                            │
┌────────────────────┐      │
│  Azure Blob        │  ◄───┘
│  (5TB data)        │
│  🔄 Syncing        │
└────────────────────┘

Phase 2: Validation
┌────────────────────┐
│  AWS S3            │
│  ✅ Active         │
└────────────────────┘

┌────────────────────┐
│  Azure Blob        │
│  ✅ Validated      │
│  (Test queries)    │
└────────────────────┘

Phase 3: Cutover
┌────────────────────────────┐
│  AWS (Web App)             │
│                            │
│  Change config:            │
│  data_service_url =        │
│    azure.infrar.io ◄───────┼─── Update registry
└────────────────────────────┘

┌────────────────────┐
│  AWS S3            │
│  ⚠️  Deprecated    │
└────────────────────┘

┌────────────────────┐
│  Azure Blob        │
│  ✅ Active         │
└────────────────────┘

Phase 4: Cleanup (after stability)
┌────────────────────┐
│  AWS S3            │
│  🗑️ Deleted        │
└────────────────────┘

┌────────────────────┐
│  Azure Blob        │
│  ✅ Active         │
└────────────────────┘
```

### Data Synchronization

**Approaches**:

1. **One-time copy** (simplest):
```bash
# Infrar executes
aws s3 sync s3://aws-bucket/ /tmp/data/
az storage blob upload-batch --source /tmp/data/ --destination azure-container
```

2. **Continuous sync** (for large datasets):
```bash
# Infrar sets up continuous replication
rclone sync s3:aws-bucket azure:azure-container --progress
```

3. **Provider-native tools**:
- AWS DataSync → Azure
- Azure Data Factory pull from S3
- GCP Transfer Service

**Infrar orchestrates**:
```python
class DataMigrator:
    def migrate_data(self, source, destination, strategy='one-time'):
        if strategy == 'one-time':
            # Copy all data at once
            self.copy_data(source, destination)

        elif strategy == 'continuous':
            # Set up ongoing replication
            replication_job = self.setup_replication(source, destination)

            # Monitor until caught up
            while not replication_job.is_synchronized():
                time.sleep(60)

            # Stop replication after cutover
            self.stop_replication(replication_job)

        # Validate data integrity
        self.validate_data(source, destination)
```

## Cost Tracking for Multi-Cloud

### Challenge

Understanding total costs when services span multiple providers.

### Infrar Solution

```python
# Cost tracking per capability
cost_breakdown = {
    'web_application': {
        'provider': 'aws',
        'region': 'us-east-1',
        'monthly_cost': 320.00,
        'breakdown': {
            'compute': 250.00,
            'networking': 45.00,
            'storage': 25.00
        }
    },
    'data_intelligence': {
        'provider': 'azure',
        'region': 'eastus',
        'monthly_cost': 340.00,
        'breakdown': {
            'compute': 200.00,
            'storage': 115.00,
            'networking': 25.00
        }
    },
    'background_workers': {
        'provider': 'gcp',
        'region': 'us-central1',
        'monthly_cost': 65.00,
        'breakdown': {
            'compute': 50.00,
            'storage': 10.00,
            'networking': 5.00
        }
    },
    'cross_cloud_networking': {
        'total': 45.00,
        'breakdown': {
            'aws_to_azure_egress': 25.00,  # 278GB @ $0.09/GB
            'aws_to_gcp_egress': 15.00,     # 125GB @ $0.12/GB
            'azure_to_aws_egress': 5.00     # 57GB @ $0.087/GB
        }
    }
}

total_monthly_cost = 770.00  # Including cross-cloud networking
```

**Infrar Dashboard**:
```
Monthly Cost Breakdown:
┌──────────────────────────────────────────┐
│ Web Application (AWS)        $320  41%  │
│ Data Intelligence (Azure)    $340  44%  │
│ Background Workers (GCP)     $65    8%  │
│ Cross-Cloud Networking       $45    6%  │
├──────────────────────────────────────────┤
│ Total                        $770  100% │
└──────────────────────────────────────────┘

Optimization Suggestions:
⚠️  High data transfer between AWS and Azure (278GB/month)
    Consider:
    - Caching frequently accessed data
    - Batch requests to reduce calls
    - Move data-intensive workload to same provider

💡 Savings opportunity: $25/month
```

## Monitoring Multi-Cloud

### Centralized Observability

Infrar aggregates logs, metrics, and traces from all providers:

```
┌─────────────────────────────────────────┐
│     Infrar Observability Platform       │
│                                         │
│  Logs  │  Metrics  │  Traces  │  Alerts│
└────┬────────┬────────┬──────────┬───────┘
     │        │        │          │
     ↓        ↓        ↓          ↓
┌─────────────────────────────────────────┐
│          Log Aggregation                │
│  (CloudWatch + Stackdriver + Monitor)   │
└─────────────────────────────────────────┘

Example Query:
"Show all requests from web-app to data-service that took > 1s"

Results:
┌──────────────────────────────────────────────┐
│ Timestamp       │ Source    │ Dest   │ Time  │
├──────────────────────────────────────────────┤
│ 2024-01-01 10:15│ AWS ECS   │ Azure  │ 1.2s  │
│ 2024-01-01 10:16│ AWS ECS   │ Azure  │ 1.5s  │
│ 2024-01-01 10:17│ AWS ECS   │ Azure  │ 1.8s  │
└──────────────────────────────────────────────┘

Insight: High latency for cross-cloud calls.
Suggestion: Enable VPN or move services closer.
```

### Distributed Tracing

Track requests across clouds:

```
Request: GET /dashboard

Trace:
1. Web App (AWS)          [50ms]
   └─> GET /api/sales

2. Data Service (Azure)   [1200ms] ⚠️
   ├─> Query Synapse      [1150ms]
   └─> Format response    [50ms]

3. Web App (AWS)          [100ms]
   └─> Render dashboard

Total: 1350ms

Bottleneck: Cross-cloud latency (AWS → Azure)
Recommendation: Cache query results for 5 minutes
Potential improvement: 1350ms → 150ms (89% faster)
```

## Security Considerations

### Network Security

```
┌─────────────────────────────────────────┐
│          Security Layers                │
│                                         │
│  1. Encryption in Transit (TLS 1.3)    │
│  2. Authentication (JWT tokens)         │
│  3. Authorization (RBAC)                │
│  4. Network isolation (VPC/VNet)        │
│  5. Firewall rules (allow-list)         │
│  6. DDoS protection                     │
│  7. Audit logging                       │
└─────────────────────────────────────────┘
```

### Least Privilege

Each service only has access to what it needs:

```yaml
# Web Application permissions
permissions:
  data-intelligence:
    - read:data
    - write:query
  # Cannot access background-workers

# Background Workers permissions
permissions:
  data-intelligence:
    - write:data
  # Cannot access web-application
```

## Failure Handling

### Cross-Cloud Resilience

```python
# Retry logic for cross-cloud calls
@retry(max_attempts=3, backoff=exponential)
def query_data_service(query):
    try:
        response = requests.post(
            infrar.registry.get_endpoint('data-intelligence') + '/query',
            json={'query': query},
            timeout=30
        )
        return response.json()

    except requests.Timeout:
        # Try alternative endpoint if available
        if infrar.registry.has_fallback('data-intelligence'):
            fallback_url = infrar.registry.get_fallback_endpoint('data-intelligence')
            response = requests.post(
                fallback_url + '/query',
                json={'query': query},
                timeout=30
            )
            return response.json()

        raise
```

### Circuit Breaker

Prevent cascading failures:

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=60)
def call_azure_service():
    # If Azure service fails 5 times, circuit opens
    # No more calls for 60 seconds (give it time to recover)
    # Then try again (half-open state)
    response = requests.post('https://azure-service...')
    return response
```

## Future Enhancements

### 1. Smart Routing

Route requests based on performance:

```python
# Infrar automatically routes to fastest endpoint
if latency_to_azure < latency_to_aws:
    use_azure_endpoint()
else:
    use_aws_endpoint()
```

### 2. Data Locality

Automatically place data close to compute:

```python
# If web app is on AWS, cache data in AWS S3
# Even if primary data is in Azure
infrar.data.cache_strategy = 'locality-optimized'
```

### 3. Cost-Optimized Routing

Route based on cost:

```python
# Use cheaper egress path
if aws_to_gcp_cost < aws_to_azure_cost:
    route_via_gcp()
```

## Conclusion

Multi-cloud orchestration is complex, but Infrar handles:
- ✅ Service discovery across providers
- ✅ Authentication and authorization
- ✅ Data migration
- ✅ Cost tracking and optimization
- ✅ Centralized monitoring
- ✅ Security and compliance

Users get the benefits of multi-cloud (cost optimization, best-of-breed) without the operational complexity.

Next: [Concepts - What are Capabilities?](../concepts/capabilities.md)
