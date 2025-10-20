# Capability Catalog

## Overview

The Capability Catalog is the knowledge base that maps functional infrastructure requirements to provider-specific implementations. It defines what each capability provides, how it's measured, and how much it costs across different cloud providers.

## Core Concept

Instead of thinking "I need ECS and RDS", users think "I need a web application and a relational database".

```
User Need (Functional) → Capability → Implementations (Provider-Specific)

"Web Application" → web_application → {
    AWS: ECS Fargate + ALB + VPC
    GCP: Cloud Run + Cloud Load Balancing
    Azure: Container Apps + Application Gateway
}
```

## Capability Schema

Each capability is defined using a structured YAML format:

```yaml
# capabilities/web_application/definition.yaml

name: web_application
display_name: "Web Application"
category: compute
description: "Stateless containerized web application with load balancing and auto-scaling"

# What this capability provides functionally
provides:
  - http_server
  - https_termination
  - auto_scaling
  - health_checks
  - zero_downtime_deployment

# How it's measured (for cost calculation)
metrics:
  - name: cpu_units
    type: numeric
    unit: vcpu
    description: "Number of virtual CPU cores"

  - name: memory_mb
    type: numeric
    unit: mb
    description: "Memory in megabytes"

  - name: request_count_month
    type: numeric
    unit: requests
    description: "HTTP requests per month"

  - name: data_transfer_gb_month
    type: numeric
    unit: gb
    description: "Outbound data transfer per month"

  - name: instance_hours_month
    type: numeric
    unit: hours
    description: "Total instance running hours per month"

# User-facing configuration
configuration:
  - name: container_image
    type: string
    required: true
    description: "Docker image to deploy"

  - name: container_port
    type: integer
    required: true
    default: 8080
    description: "Port the container listens on"

  - name: environment_variables
    type: map
    required: false
    description: "Environment variables for the container"

  - name: min_instances
    type: integer
    required: true
    default: 1
    description: "Minimum number of instances"

  - name: max_instances
    type: integer
    required: true
    default: 10
    description: "Maximum number of instances"

# Provider implementations
implementations:
  aws_ecs_fargate:
    provider: aws
    services:
      - ecs
      - alb
      - vpc
    resources:
      - ecs_cluster
      - ecs_task_definition
      - ecs_service
      - alb
      - target_group
      - vpc
      - subnets
      - security_groups

    # Cost model
    cost_model:
      formula: |
        (cpu_units * 0.04048 * instance_hours_month) +
        (memory_mb / 1024 * 0.004445 * instance_hours_month) +
        (alb_hours_month * 0.0225) +
        (alb_lcu_month * 0.008) +
        (data_transfer_gb_month * 0.09)
      currency: USD
      variables:
        alb_hours_month: 730  # Always running
        alb_lcu_month: "request_count_month / 25"  # 25 requests per LCU

    # Feature coverage
    capabilities_coverage:
      http_server: full
      https_termination: full
      auto_scaling: full
      health_checks: full
      zero_downtime_deployment: full

    # Provider-specific configuration
    provider_config:
      task_cpu: "cpu_units * 1024"  # ECS uses CPU units
      task_memory: "memory_mb"
      desired_count: "min_instances"
      max_count: "max_instances"

  gcp_cloud_run:
    provider: gcp
    services:
      - cloud_run
      - cloud_load_balancing

    resources:
      - cloud_run_service
      - load_balancer
      - backend_service

    cost_model:
      formula: |
        (cpu_units * 0.00002400 * cpu_time_seconds_month) +
        (memory_mb / 1024 * 0.00000250 * memory_time_seconds_month) +
        (request_count_month * 0.00000040) +
        (data_transfer_gb_month * 0.12)
      currency: USD
      variables:
        # Cloud Run charges per 100ms, assumes 200ms avg request
        cpu_time_seconds_month: "request_count_month * 0.2"
        memory_time_seconds_month: "request_count_month * 0.2"

    capabilities_coverage:
      http_server: full
      https_termination: full
      auto_scaling: full  # Including scale to zero
      health_checks: full
      zero_downtime_deployment: full

    provider_config:
      cpu_limit: "cpu_units"
      memory_limit: "memory_mb Mi"
      min_instances: "min_instances"  # Can be 0 for scale to zero
      max_instances: "max_instances"

  azure_container_apps:
    provider: azure
    services:
      - container_apps
      - application_gateway

    resources:
      - container_app
      - container_app_environment
      - application_gateway

    cost_model:
      formula: |
        (cpu_units * 0.000015 * cpu_time_seconds_month) +
        (memory_mb / 1024 * 0.0000016 * memory_time_seconds_month) +
        (request_count_month * 0.0000004) +
        (data_transfer_gb_month * 0.087)
      currency: USD
      variables:
        cpu_time_seconds_month: "instance_hours_month * 3600"
        memory_time_seconds_month: "instance_hours_month * 3600"

    capabilities_coverage:
      http_server: full
      https_termination: full
      auto_scaling: full
      health_checks: full
      zero_downtime_deployment: full

    provider_config:
      cpu: "cpu_units"
      memory: "memory_mb / 1024 Gi"
      min_replicas: "min_instances"
      max_replicas: "max_instances"

# Cross-cloud interfaces (for multi-cloud scenarios)
cross_cloud_interfaces:
  - type: http_endpoint
    protocol: https
    authentication: [api_key, oauth2, iam]
    egress_cost_per_gb:
      aws: 0.09
      gcp: 0.12
      azure: 0.087

# Migration compatibility
migration:
  state_type: stateless
  data_migration_required: false
  downtime_required: false
  blue_green_supported: true
```

## Capability Categories

### Compute

#### web_application
- **Purpose**: Serve HTTP/HTTPS traffic
- **Use Cases**: Web apps, REST APIs, SSR applications
- **Implementations**: ECS, Cloud Run, Container Apps

#### api_service
- **Purpose**: Backend API with auto-scaling
- **Use Cases**: REST APIs, GraphQL, gRPC
- **Implementations**: API Gateway + Lambda, Cloud Functions, Azure Functions

#### background_worker
- **Purpose**: Async job processing
- **Use Cases**: Queue consumers, batch jobs, scheduled tasks
- **Implementations**: ECS (no ALB), Cloud Run Jobs, Container Apps Jobs

#### container_orchestrator
- **Purpose**: Full Kubernetes orchestration
- **Use Cases**: Complex microservices, existing K8s apps
- **Implementations**: EKS, GKE, AKS

### Storage

#### object_storage
- **Purpose**: Store unstructured data (files, images, backups)
- **Use Cases**: File uploads, static assets, backups
- **Implementations**: S3, Cloud Storage, Blob Storage

#### relational_database
- **Purpose**: Managed SQL database
- **Use Cases**: Transactional data, relational queries
- **Implementations**: RDS (PostgreSQL/MySQL), Cloud SQL, Azure SQL

#### cache
- **Purpose**: In-memory caching
- **Use Cases**: Session storage, cache layer
- **Implementations**: ElastiCache (Redis), Memorystore, Azure Cache

### Data Intelligence

#### data_warehouse
- **Purpose**: SQL analytics on large datasets
- **Use Cases**: Business intelligence, reporting, analytics
- **Implementations**: Athena, BigQuery, Synapse

#### etl_pipeline
- **Purpose**: Extract, transform, load data
- **Use Cases**: Data integration, transformation
- **Implementations**: Glue, Dataflow, Data Factory

### Networking

#### vpc
- **Purpose**: Private network
- **Use Cases**: Network isolation, security
- **Implementations**: VPC, VPC, VNet

#### cdn
- **Purpose**: Content delivery network
- **Use Cases**: Static asset delivery, global distribution
- **Implementations**: CloudFront, Cloud CDN, Azure CDN

### Messaging

#### message_queue
- **Purpose**: Asynchronous message queue
- **Use Cases**: Job queues, event processing
- **Implementations**: SQS, Pub/Sub (pull), Service Bus

#### event_bus
- **Purpose**: Pub/sub messaging
- **Use Cases**: Event-driven architecture, fan-out
- **Implementations**: SNS, Pub/Sub, Event Grid

## Example: Data Intelligence Capability

```yaml
# capabilities/data_intelligence/definition.yaml

name: data_intelligence
display_name: "Data Intelligence & Analytics"
category: data
description: "SQL analytics, ETL orchestration, and data cataloging for large datasets"

provides:
  - sql_query_engine
  - etl_orchestration
  - data_catalog
  - schema_inference
  - data_transformation

metrics:
  - name: data_scanned_tb_month
    type: numeric
    unit: tb
    description: "Terabytes of data scanned per month"

  - name: data_stored_tb
    type: numeric
    unit: tb
    description: "Terabytes of data stored"

  - name: query_count_month
    type: numeric
    unit: queries
    description: "Number of SQL queries per month"

  - name: etl_job_hours_month
    type: numeric
    unit: hours
    description: "Total ETL job runtime hours per month"

configuration:
  - name: data_format
    type: enum
    options: [parquet, csv, json, avro]
    required: true
    description: "Primary data format"

  - name: query_frequency
    type: enum
    options: [realtime, hourly, daily, weekly]
    required: true
    description: "How often queries are run"

  - name: concurrent_queries
    type: integer
    required: true
    default: 10
    description: "Maximum concurrent queries"

implementations:
  aws_athena_glue:
    provider: aws
    services:
      - athena
      - glue
      - s3

    cost_model:
      formula: |
        (data_scanned_tb_month * 5.00) +
        (etl_job_hours_month * 0.44) +
        (data_stored_tb * 0.023) +
        (glue_data_catalog_objects * 1.00 / 1_000_000)
      currency: USD
      variables:
        glue_data_catalog_objects: 1000  # Typical

    capabilities_coverage:
      sql_query_engine: full
      etl_orchestration: full
      data_catalog: full
      schema_inference: full
      data_transformation: full

    provider_config:
      database_name: "analytics_db"
      workgroup_name: "primary"
      output_location: "s3://query-results/"

  gcp_bigquery_dataflow:
    provider: gcp
    services:
      - bigquery
      - dataflow
      - cloud_storage

    cost_model:
      formula: |
        (data_scanned_tb_month * 5.00) +
        (data_stored_tb * 0.020) +
        (etl_job_hours_month * 4 * 0.036) +
        (data_streaming_gb_month * 0.010)
      currency: USD
      variables:
        data_streaming_gb_month: 0  # Batch only for MVP

    capabilities_coverage:
      sql_query_engine: full
      etl_orchestration: full
      data_catalog: full
      schema_inference: full
      data_transformation: full

    provider_config:
      dataset_name: "analytics_dataset"
      location: "US"

  azure_synapse:
    provider: azure
    services:
      - synapse_analytics
      - data_factory
      - blob_storage

    cost_model:
      formula: |
        (data_scanned_tb_month * 6.00) +
        (data_stored_tb * 0.018) +
        (etl_job_hours_month * 0.52) +
        (pipeline_activities * 0.001)
      currency: USD
      variables:
        pipeline_activities: "etl_job_hours_month * 10"  # Estimate

    capabilities_coverage:
      sql_query_engine: full
      etl_orchestration: full
      data_catalog: partial  # Requires Azure Purview
      schema_inference: full
      data_transformation: full

    provider_config:
      workspace_name: "analytics_workspace"
      sql_pool_type: "serverless"

# Cross-cloud data access
cross_cloud_interfaces:
  - type: data_access
    protocols: [https, s3_api]
    authentication: [iam, sas_token, service_account]
    egress_cost_per_gb:
      aws: 0.09
      gcp: 0.12
      azure: 0.087

migration:
  state_type: stateful
  data_migration_required: true
  data_migration_strategy: [export_import, replication, etl]
  downtime_required: false  # Can run parallel
  blue_green_supported: true
```

## Cost Calculation Engine

### How Costs Are Calculated

```python
class CostCalculator:
    def __init__(self, capability_catalog):
        self.catalog = capability_catalog

    def calculate_cost(self, capability_name, metrics, provider):
        """
        Calculate monthly cost for a capability on a specific provider.

        Args:
            capability_name: e.g., "web_application"
            metrics: Dict of metric values, e.g., {"cpu_units": 2, "memory_mb": 4096, ...}
            provider: "aws", "gcp", or "azure"

        Returns:
            Cost in USD per month
        """
        capability = self.catalog.get_capability(capability_name)

        # Find implementation for provider
        implementation = None
        for impl_name, impl in capability['implementations'].items():
            if impl['provider'] == provider:
                implementation = impl
                break

        if not implementation:
            raise ValueError(f"No implementation for {capability_name} on {provider}")

        # Get cost model
        cost_model = implementation['cost_model']
        formula = cost_model['formula']
        variables = cost_model.get('variables', {})

        # Build evaluation context
        context = metrics.copy()

        # Resolve variables (some may be formulas themselves)
        for var_name, var_value in variables.items():
            if isinstance(var_value, str):
                # It's a formula, evaluate it
                context[var_name] = eval(var_value, {"__builtins__": {}}, context)
            else:
                context[var_name] = var_value

        # Evaluate cost formula
        cost = eval(formula, {"__builtins__": {}}, context)

        return round(cost, 2)
```

### Example Cost Calculation

```python
calculator = CostCalculator(capability_catalog)

# User metrics
metrics = {
    'cpu_units': 2,
    'memory_mb': 4096,
    'request_count_month': 10_000_000,  # 10M requests
    'data_transfer_gb_month': 500,       # 500GB egress
    'instance_hours_month': 730          # Always running (1 instance)
}

# Calculate for each provider
aws_cost = calculator.calculate_cost('web_application', metrics, 'aws')
gcp_cost = calculator.calculate_cost('web_application', metrics, 'gcp')
azure_cost = calculator.calculate_cost('web_application', metrics, 'azure')

print(f"AWS (ECS Fargate): ${aws_cost}/month")
print(f"GCP (Cloud Run): ${gcp_cost}/month")
print(f"Azure (Container Apps): ${azure_cost}/month")

# Output:
# AWS (ECS Fargate): $127.45/month
# GCP (Cloud Run): $98.20/month
# Azure (Container Apps): $105.80/month
```

## Capability Matching

### From Resources to Capabilities (Discovery)

When scanning existing infrastructure:

```python
def map_resources_to_capability(resources):
    """
    Map discovered cloud resources to capabilities.

    Args:
        resources: List of cloud resources with types and configurations

    Returns:
        List of matched capabilities with confidence scores
    """

    # Example: AWS resources
    if has_resources(resources, ['ecs_cluster', 'ecs_service', 'alb']):
        return {
            'capability': 'web_application',
            'implementation': 'aws_ecs_fargate',
            'confidence': 'high',
            'resources': filter_resources(resources, ['ecs', 'alb', 'vpc'])
        }

    if has_resources(resources, ['athena', 'glue', 's3']):
        return {
            'capability': 'data_intelligence',
            'implementation': 'aws_athena_glue',
            'confidence': 'high',
            'resources': filter_resources(resources, ['athena', 'glue', 's3'])
        }

    # More matching logic...
```

### From Requirements to Implementation (Selection)

When deploying new infrastructure:

```python
def recommend_implementation(capability_name, requirements, preferences):
    """
    Recommend best implementation based on requirements and preferences.

    Args:
        capability_name: e.g., "web_application"
        requirements: User's technical requirements
        preferences: User's preferences (cost priority, region, etc.)

    Returns:
        Ranked list of implementations
    """

    capability = capability_catalog.get_capability(capability_name)
    implementations = capability['implementations']

    recommendations = []

    for impl_name, impl in implementations.items():
        # Calculate cost
        cost = calculate_cost(capability_name, requirements['metrics'], impl['provider'])

        # Check feature coverage
        coverage_score = calculate_coverage_score(requirements['features'], impl['capabilities_coverage'])

        # Check regional availability
        region_available = impl['provider'] in preferences.get('allowed_providers', [])

        # Calculate overall score
        score = (
            (1 - cost / max_cost) * preferences['cost_weight'] +
            coverage_score * preferences['features_weight'] +
            (1 if region_available else 0) * preferences['region_weight']
        )

        recommendations.append({
            'implementation': impl_name,
            'provider': impl['provider'],
            'cost': cost,
            'coverage_score': coverage_score,
            'overall_score': score
        })

    # Sort by score
    recommendations.sort(key=lambda x: x['overall_score'], reverse=True)

    return recommendations
```

## Extending the Catalog

### Adding New Capabilities

1. Create capability definition YAML
2. Define metrics and configuration
3. Add implementations for each provider
4. Define cost models
5. Test cost calculations
6. Add to catalog

### Adding New Implementations

To add a new implementation to existing capability:

```yaml
# In capabilities/web_application/definition.yaml

implementations:
  # ... existing implementations ...

  digitalocean_app_platform:
    provider: digitalocean
    services:
      - app_platform

    cost_model:
      formula: |
        (cpu_units * memory_mb / 1024 * 12.00) +
        (data_transfer_gb_month * 0.01)
      currency: USD

    capabilities_coverage:
      http_server: full
      https_termination: full
      auto_scaling: limited  # Only vertical scaling
      health_checks: full
      zero_downtime_deployment: full

    provider_config:
      instance_size: "professional-xs"  # Map from cpu/memory
```

## Catalog Versioning

As cloud provider pricing and features change:

```yaml
# capabilities/web_application/versions.yaml

versions:
  - version: "1.0.0"
    date: "2024-01-01"
    changes: "Initial version"

  - version: "1.1.0"
    date: "2024-06-01"
    changes: "Updated AWS ECS pricing, added GCP Cloud Run scale-to-zero"

  - version: "1.2.0"
    date: "2024-12-01"
    changes: "Added Azure Container Apps, updated data transfer costs"
```

## Conclusion

The Capability Catalog is the foundation of Infrar's intelligence:
- Provides functional abstraction over technical resources
- Enables accurate cost comparison
- Guides implementation selection
- Supports discovery of existing infrastructure
- Maintains up-to-date provider information

Next: [Multi-Cloud Orchestration](multi-cloud-orchestration.md) - How to run workloads across multiple providers
