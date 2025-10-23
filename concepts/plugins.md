# Provider Plugin System

## Overview

Infrar uses a provider-based plugin architecture where all cloud provider-specific logic, configuration, and templates are externalized into the **infrar-plugins** repository. The platform code remains 100% provider-agnostic, loading all provider-specific behavior dynamically from plugin YAML files and templates.

## Architecture Principles

1. **Provider-First Organization**: Plugins are organized by provider (aws/gcp/azure), not by capability
2. **Template-Driven**: All terraform code generation uses templates, not hardcoded strings
3. **YAML Configuration**: All metadata, rules, and settings defined in YAML
4. **Zero Platform Coupling**: Adding a new provider requires zero changes to platform code

## Plugin Directory Structure

```
infrar-plugins/
└── providers/
    ├── aws/
    │   ├── provider.yaml               # Provider metadata
    │   ├── regions.yaml                # Available regions
    │   ├── provider-defaults.yaml      # Default settings (region, labels, etc.)
    │   ├── pricing-metadata.yaml       # Cost estimation data
    │   ├── variables-ui-metadata.yaml  # Frontend form configuration
    │   │
    │   ├── authentication/             # Authentication methods
    │   │   └── {method-name}/
    │   │       ├── method.yaml         # Method metadata
    │   │       ├── schema.yaml         # Credential schema
    │   │       ├── handler.yaml        # Setup instructions
    │   │       └── ui-instructions.yaml # Frontend instructions
    │   │
    │   ├── services/                   # Service capabilities
    │   │   ├── compute/
    │   │   │   └── {service-name}/
    │   │   │       ├── service.yaml    # Service metadata
    │   │   │       ├── defaults.yaml   # Default configuration
    │   │   │       ├── terraform/      # Terraform modules
    │   │   │       │   └── main.tf
    │   │   │       └── transform/      # Code transformation rules
    │   │   │           └── rules.yaml
    │   │   ├── storage/
    │   │   ├── database/
    │   │   └── messaging/
    │   │
    │   ├── terraform-config/           # Provider-level terraform templates
    │   │   ├── provider-block.tf.tmpl
    │   │   ├── backend.tf.tmpl
    │   │   └── variables.tf.tmpl
    │   │
    │   └── orchestration/              # Bootstrap and setup scripts
    │       └── bootstrap-script.sh.tmpl
    │
    ├── gcp/                            # Same structure as AWS
    └── azure/                          # Same structure as AWS
```

## Key Plugin Components

### 1. Provider Metadata (`provider.yaml`)

Defines basic provider information:

```yaml
provider:
  id: gcp
  name: Google Cloud Platform
  display_name: GCP
  description: "Google Cloud Platform provider"
  version: 1.0.0

  supported_capabilities:
    - compute
    - storage
    - database
    - messaging

  terraform_provider:
    source: "hashicorp/google"
    version: "~> 5.0"
```

### 2. Provider Defaults (`provider-defaults.yaml`)

Default configuration values for the provider:

```yaml
provider_defaults:
  id: gcp-defaults
  version: 1.0.0
  provider: gcp

  # Default region for resources
  default_region: us-central1

  # Default compute configuration
  compute:
    runtime: python311
    memory_size: 512
    timeout: 30
    min_instances: 0
    max_instances: 10

  # Default storage configuration
  storage:
    location: US
    versioning_enabled: false

  # Label/Tag configuration
  label_format: lowercase_underscore  # or mixed_case
  label_key: labels                   # or tags
```

### 3. Authentication Methods (`authentication/{method}/`)

Each authentication method is a plugin with four files:

#### `method.yaml` - Method Metadata
```yaml
authentication_method:
  id: service-account-json
  name: Service Account JSON
  display_name: "Service Account (JSON Key)"
  description: "Authenticate using a GCP service account JSON key file"
  recommended: true

  security_level: high
  setup_complexity: medium
```

#### `schema.yaml` - Credential Schema
```yaml
credential_schema:
  type: object
  required:
    - service_account_json
  properties:
    service_account_json:
      type: string
      format: json
      description: "Complete JSON key file content"
      sensitive: true
```

#### `handler.yaml` - Setup Instructions
```yaml
handler:
  setup_type: credential_file

  environment_variables:
    GOOGLE_APPLICATION_CREDENTIALS:
      source: file_path
      description: "Path to service account JSON file"

  file_setup:
    - name: service_account.json
      content_source: service_account_json
      permissions: "0600"
```

#### `ui-instructions.yaml` - Frontend UI Instructions
```yaml
ui_instructions:
  title: "GCP Service Account Setup"
  estimated_time: "5 minutes"

  steps:
    - number: 1
      title: "Open GCP Console"
      description: "Navigate to IAM & Admin → Service Accounts"
      action_url: "https://console.cloud.google.com/iam-admin/serviceaccounts"
```

### 4. Service Capabilities (`services/{category}/{service}/`)

Each service capability has:

#### `service.yaml` - Service Metadata
```yaml
service:
  id: cloud-storage
  name: "Cloud Storage"
  display_name: "Google Cloud Storage"
  provider: gcp
  category: storage
  capability_type: storage

  description: "Object storage service for GCP"
  documentation_url: "https://cloud.google.com/storage/docs"

  # Required GCP APIs
  provider_apis:
    - name: "storage-api.googleapis.com"
      display_name: "Cloud Storage API"
      required: true
```

#### `terraform/main.tf` - Terraform Module
```hcl
# GCS Bucket Resource
resource "google_storage_bucket" "main" {
  name          = var.bucket_name
  location      = var.region
  force_destroy = var.force_destroy

  labels = var.labels

  versioning {
    enabled = var.versioning_enabled
  }
}
```

#### `transform/rules.yaml` - Code Transformation Rules
```yaml
transformations:
  - pattern: "infrar.storage.upload"
    target:
      provider: gcp
      service: cloud-storage

    imports:
      - "from google.cloud import storage"

    template: |
      storage_client = storage.Client()
      bucket = storage_client.bucket({{bucket}})
      blob = bucket.blob({{destination}})
      blob.upload_from_filename({{source}})
```

### 5. Terraform Configuration Templates (`terraform-config/`)

Provider-level terraform templates:

#### `provider-block.tf.tmpl`
```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

## How the Plugin System Works

### 1. Platform Startup
```
Platform starts
    ↓
Load all providers from plugins/providers/
    ↓
Parse provider.yaml for each provider
    ↓
Register authentication methods
    ↓
Register service capabilities
    ↓
Build capability catalog
```

### 2. Project Transformation
```
User code with infrar SDK
    ↓
Detect capabilities used (storage, compute, etc.)
    ↓
Load service capability plugin for selected provider
    ↓
Apply transformation rules from transform/rules.yaml
    ↓
Generate native provider SDK code
```

### 3. Infrastructure Generation
```
Capabilities detected
    ↓
For each capability, load service terraform module
    ↓
Render provider-block.tf.tmpl with provider
    ↓
Render variables.tf.tmpl with defaults
    ↓
Combine all terraform files
    ↓
Generate tfvars from provider-defaults.yaml
```

### 4. Deployment
```
Load authentication method for provider
    ↓
Setup credentials using handler.yaml instructions
    ↓
Execute bootstrap script from orchestration/
    ↓
Run terraform init/plan/apply
    ↓
Deploy application
```

## Adding a New Provider

To add a new cloud provider (e.g., DigitalOcean):

1. **Create Provider Directory**
   ```bash
   mkdir -p providers/digitalocean
   ```

2. **Create Provider Metadata**
   - `provider.yaml` - Basic provider info
   - `regions.yaml` - Available regions
   - `provider-defaults.yaml` - Default settings
   - `pricing-metadata.yaml` - Cost data

3. **Add Authentication Method**
   ```bash
   mkdir -p providers/digitalocean/authentication/api-token
   ```
   Create method.yaml, schema.yaml, handler.yaml, ui-instructions.yaml

4. **Add Service Capabilities**
   ```bash
   mkdir -p providers/digitalocean/services/storage/spaces
   ```
   Create service.yaml, defaults.yaml, terraform/main.tf, transform/rules.yaml

5. **Add Terraform Templates**
   ```bash
   mkdir -p providers/digitalocean/terraform-config
   ```
   Create provider-block.tf.tmpl, backend.tf.tmpl, variables.tf.tmpl

6. **Add Orchestration**
   ```bash
   mkdir -p providers/digitalocean/orchestration
   ```
   Create bootstrap-script.sh.tmpl

7. **Test**
   - Platform automatically discovers new provider
   - No platform code changes required
   - Test authentication, transformation, and deployment

## Benefits of This Architecture

✅ **Zero Platform Coupling**: Adding providers requires no platform changes
✅ **Community Contributions**: Easy for community to add providers
✅ **Maintainability**: Provider logic isolated in plugins
✅ **Testability**: Each plugin can be tested independently
✅ **Flexibility**: Providers can evolve independently
✅ **Consistency**: All providers follow same structure

## Migration from Old Architecture

The previous architecture had:
- Hardcoded provider logic in platform code
- Switch statements on provider names
- Provider-specific functions scattered across codebase

The current architecture has:
- **All** provider-specific logic in plugins
- Template-based generation
- Dynamic plugin loading
- Zero switch statements on provider names in platform code

## Next Steps

- See [CONTRIBUTING.md](../CONTRIBUTING.md) for detailed guide on adding a new provider
- See [Architecture Overview](../architecture/overview.md) for system architecture
- See [Repository Structure](../architecture/repository-structure.md) for codebase organization
