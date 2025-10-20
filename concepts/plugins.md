# Plugin System

## Overview

Infrar uses a plugin-based architecture where each functional domain is implemented as a separate plugin. This provides modularity, maintainability, and allows gradual feature expansion.

## Plugin Architecture

```
infrar (core platform)
├── infrar-core (compute, networking - infrastructure only)
├── infrar-storage (object storage, file operations)
├── infrar-database (relational databases)
├── infrar-messaging (queues, pub/sub)
├── infrar-data (analytics, ETL, warehousing)
└── infrar-observability (logging, monitoring, tracing)
```

Each plugin provides:
1. **SDK**: User-facing API for application code
2. **Transformation Rules**: How to convert SDK calls to provider code
3. **OpenTofu Modules**: Infrastructure provisioning templates
4. **Cost Models**: Pricing calculations per provider

## Plugin Structure

```
infrar-storage/
├── sdk/                          # User-facing SDK
│   ├── __init__.py
│   ├── operations.py             # upload, download, list, delete
│   └── exceptions.py             # StorageError, etc.
│
├── transformations/              # Code transformation rules
│   ├── rules.yaml                # Transformation definitions
│   ├── aws.py                    # AWS-specific transformations
│   ├── gcp.py                    # GCP-specific transformations
│   └── azure.py                  # Azure-specific transformations
│
├── modules/                      # OpenTofu modules
│   ├── aws/
│   │   ├── s3_bucket/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── ...
│   ├── gcp/
│   │   └── cloud_storage_bucket/
│   └── azure/
│       └── blob_container/
│
├── cost_models/                  # Pricing data
│   ├── aws.yaml
│   ├── gcp.yaml
│   └── azure.yaml
│
└── tests/
    ├── test_transformations.py
    ├── test_sdk.py
    └── test_cost_models.py
```

## Core Plugins

### infrar-core (Infrastructure)

**Purpose**: Foundational compute and networking (no code transformation needed)

**Capabilities**:
- `web_application`: Containerized web apps with load balancing
- `api_service`: Backend API services
- `background_worker`: Async job processors
- `container_orchestrator`: Kubernetes clusters

**User Interface** (configuration, not code):
```yaml
# project.yaml
capabilities:
  - name: my-webapp
    type: web_application
    config:
      container_image: myorg/app:latest
      cpu: 2
      memory: 4096
```

**No SDK calls in code** - this is pure infrastructure.

### infrar-storage (Object Storage)

**Purpose**: File and object storage operations

**SDK**:
```python
from infrar.storage import upload, download, list_objects, delete

# Upload file
upload(bucket='my-bucket', source='local.txt', destination='remote.txt')

# Download file
download(bucket='my-bucket', source='remote.txt', destination='local.txt')

# List objects
files = list_objects(bucket='my-bucket', prefix='data/')

# Delete object
delete(bucket='my-bucket', path='old-file.txt')
```

**Transformations**:
- AWS → `boto3.client('s3')`
- GCP → `google.cloud.storage`
- Azure → `azure.storage.blob`

### infrar-database (Relational Databases)

**Purpose**: SQL database operations

**SDK**:
```python
from infrar.database import query, execute, transaction

# Query
results = query(
    database='mydb',
    sql='SELECT * FROM users WHERE active = ?',
    params=[True]
)

# Execute (INSERT/UPDATE/DELETE)
execute(
    database='mydb',
    sql='INSERT INTO users (name, email) VALUES (?, ?)',
    params=['John', 'john@example.com']
)

# Transaction
with transaction(database='mydb') as tx:
    tx.execute('INSERT INTO users ...')
    tx.execute('INSERT INTO audit_log ...')
    tx.commit()
```

**Transformations**:
- AWS RDS → `boto3.client('rds-data')`  or `psycopg2`
- GCP Cloud SQL → `google.cloud.sql`
- Azure SQL → `pyodbc`

### infrar-messaging (Message Queues & Pub/Sub)

**Purpose**: Asynchronous messaging

**SDK**:
```python
from infrar.messaging import publish, subscribe, send_message, receive_messages

# Pub/Sub (topic/subscription model)
publish(topic='orders', message={'order_id': 123, 'status': 'pending'})

def handle_order(message):
    print(f"Processing order: {message['order_id']}")

subscribe(topic='orders', handler=handle_order)

# Queue (producer/consumer model)
send_message(queue='jobs', message={'job': 'process_data', 'file': 'data.csv'})

messages = receive_messages(queue='jobs', max_count=10)
for msg in messages:
    process(msg)
    msg.acknowledge()
```

**Transformations**:
- AWS → `boto3.client('sns')` / `boto3.client('sqs')`
- GCP → `google.cloud.pubsub`
- Azure → `azure.servicebus`

### infrar-data (Analytics & Data Warehousing)

**Purpose**: Large-scale data analytics

**SDK**:
```python
from infrar.data import query, run_job, catalog

# Query data warehouse
results = query(
    warehouse='analytics',
    sql='SELECT product, SUM(revenue) FROM sales GROUP BY product'
)

# Run ETL job
job = run_job(
    name='daily_aggregation',
    script='s3://scripts/aggregate.py',
    schedule='0 2 * * *'  # 2 AM daily
)

# Data catalog
tables = catalog.list_tables(database='analytics')
schema = catalog.get_schema(table='sales')
```

**Transformations**:
- AWS → `boto3.client('athena')` / `boto3.client('glue')`
- GCP → `google.cloud.bigquery`
- Azure → `azure.synapse`

## Plugin Development

### Creating a New Plugin

#### Step 1: Define SDK

```python
# infrar-cache/sdk/operations.py

def get(cache, key):
    """
    Get value from cache.

    Args:
        cache: Cache name
        key: Cache key

    Returns:
        Cached value or None
    """
    # This will be transformed away in production
    # But works in local development
    provider = os.environ.get('INFRAR_PROVIDER')

    if provider == 'aws':
        import boto3
        client = boto3.client('elasticache')
        # ... implementation

    elif provider == 'gcp':
        from google.cloud import memcache
        # ... implementation

    # etc.


def set(cache, key, value, ttl=3600):
    """Set value in cache with TTL."""
    pass


def delete(cache, key):
    """Delete key from cache."""
    pass
```

#### Step 2: Define Transformation Rules

```yaml
# infrar-cache/transformations/rules.yaml

transformations:
  infrar.cache.get:
    aws:
      imports:
        - import boto3
      template: |
        elasticache_client = boto3.client('elasticache')
        response = elasticache_client.get_cache_data(
            CacheClusterId={cache},
            Key={key}
        )
        {result_var} = response.get('Value')

    gcp:
      imports:
        - from google.cloud import memcache
      template: |
        memcache_client = memcache.Client()
        {result_var} = memcache_client.get({key})

    azure:
      imports:
        - from azure.cache.redis import RedisClient
      template: |
        redis_client = RedisClient.from_connection_string(
            os.environ['AZURE_REDIS_CONNECTION_STRING']
        )
        {result_var} = redis_client.get({key})
```

#### Step 3: Create OpenTofu Modules

```hcl
# infrar-cache/modules/aws/elasticache_redis/main.tf

resource "aws_elasticache_cluster" "cache" {
  cluster_id           = var.cache_name
  engine               = "redis"
  node_type            = var.instance_type
  num_cache_nodes      = var.num_nodes
  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name = aws_elasticache_subnet_group.cache.name
  security_group_ids = [aws_security_group.cache.id]

  tags = var.tags
}

# ... more resources ...
```

#### Step 4: Define Cost Models

```yaml
# infrar-cache/cost_models/aws.yaml

implementations:
  aws_elasticache_redis:
    cost_model:
      formula: |
        (instance_hours * instance_cost_per_hour) +
        (storage_gb * storage_cost_per_gb) +
        (data_transfer_gb * 0.09)

      instance_costs:
        cache.t3.micro: 0.017
        cache.t3.small: 0.034
        cache.t3.medium: 0.068
        cache.m5.large: 0.170
        # ... more instance types

      storage_cost_per_gb: 0.10
```

#### Step 5: Tests

```python
# infrar-cache/tests/test_transformations.py

def test_get_transformation_aws():
    source = """
from infrar.cache import get

value = get(cache='session-cache', key='user:123')
    """

    result = transform_code(source, provider='aws')

    assert 'import boto3' in result
    assert 'elasticache_client' in result
    assert 'infrar' not in result
```

## Plugin Registration

```python
# infrar/plugin_registry.py

PLUGINS = {
    'core': {
        'module': 'infrar_core',
        'version': '1.0.0',
        'provides': ['web_application', 'api_service', ...]
    },
    'storage': {
        'module': 'infrar_storage',
        'version': '1.0.0',
        'provides': ['object_storage'],
        'transformations': True  # Has code transformations
    },
    'database': {
        'module': 'infrar_database',
        'version': '1.0.0',
        'provides': ['relational_database'],
        'transformations': True
    },
    'messaging': {
        'module': 'infrar_messaging',
        'version': '1.0.0',
        'provides': ['message_queue', 'pub_sub'],
        'transformations': True
    }
}
```

## Plugin Dependencies

```yaml
# infrar-data/plugin.yaml

name: infrar-data
version: 1.0.0
description: "Analytics and data warehousing capabilities"

dependencies:
  - plugin: infrar-storage
    version: ">=1.0.0"
    reason: "Data warehouses use object storage"

  - plugin: infrar-messaging
    version: ">=1.0.0"
    reason: "ETL jobs use message queues"
    optional: true

provides:
  capabilities:
    - data_warehouse
    - etl_pipeline

  transformations:
    - infrar.data.query
    - infrar.data.run_job

  modules:
    - aws/athena_glue
    - gcp/bigquery
    - azure/synapse
```

## Provider-Specific Features

Some features only exist on specific providers:

```python
# Core feature (works everywhere)
from infrar.storage import upload

# Provider-specific feature (AWS only)
from infrar.aws.s3 import enable_versioning, set_lifecycle_policy

# Use core feature
upload(bucket='data', source='file.txt', destination='file.txt')

# Use AWS-specific feature (explicitly opt-in to AWS lock-in)
enable_versioning(bucket='data')
set_lifecycle_policy(
    bucket='data',
    policy={
        'Rules': [
            {
                'Expiration': {'Days': 90},
                'Filter': {'Prefix': 'logs/'},
                'Status': 'Enabled'
            }
        ]
    }
)
```

**Infrar warns**:
```
⚠️  Warning: Using AWS-specific feature 'enable_versioning'
    This will prevent migration to GCP or Azure
    Consider using provider-agnostic alternatives if available
```

## Plugin Versioning

```python
# User specifies plugin versions
requirements.txt:
infrar-core==1.0.0
infrar-storage==1.2.0
infrar-database==1.0.0

# Infrar ensures compatibility during transformation
if user_plugin_version != deployed_plugin_version:
    warn("Plugin version mismatch - may cause issues")
```

## Local Development with Plugins

```bash
# Install plugins for local development
pip install infrar-core infrar-storage infrar-database

# Set provider for local testing
export INFRAR_PROVIDER=aws
export AWS_PROFILE=dev

# Run application locally
python app.py

# Infrar SDK actually works (calls real AWS)
```

## Plugin Testing Strategy

### Unit Tests

Test individual transformations:
```python
def test_upload_to_s3():
    code = "upload(bucket='data', source='file.txt', destination='file.txt')"
    result = transform(code, 'aws')
    assert 's3_client.upload_file' in result
```

### Integration Tests

Test with real cloud providers:
```python
@pytest.mark.integration
@pytest.mark.aws
def test_real_upload():
    upload(bucket='test-bucket', source='/tmp/test.txt', destination='test.txt')
    # Verify file in S3
    assert file_exists_in_s3('test-bucket', 'test.txt')
```

### End-to-End Tests

Test full deployment pipeline:
```python
def test_deploy_with_storage():
    project = create_project(plugins=['infrar-storage'])
    deploy(project, provider='aws')
    # Verify app can upload files
    assert app_can_upload_files(project)
```

## Conclusion

The plugin system provides:
- ✅ **Modularity**: Each domain is independent
- ✅ **Extensibility**: Easy to add new capabilities
- ✅ **Testability**: Each plugin tested separately
- ✅ **Flexibility**: Users install only what they need
- ✅ **Evolution**: Plugins can version independently

Next: [Transformation](transformation.md) - Deep dive into compile-time transformation
