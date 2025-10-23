# Contributing to Infrar

Thank you for your interest in contributing to Infrar! This guide will help you understand how to contribute to the project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Adding a New Cloud Provider](#adding-a-new-cloud-provider)
- [Adding a New Service to an Existing Provider](#adding-a-new-service-to-an-existing-provider)
- [Plugin Development Guidelines](#plugin-development-guidelines)
- [Testing Your Changes](#testing-your-changes)
- [Submitting Changes](#submitting-changes)

## Code of Conduct

This project adheres to a code of conduct. By participating, you are expected to uphold this code. Please be respectful and constructive in all interactions.

## How Can I Contribute?

There are many ways to contribute to Infrar:

1. **Add a New Cloud Provider** - Support for DigitalOcean, Linode, Oracle Cloud, etc.
2. **Add Services to Existing Providers** - Expand AWS/GCP/Azure coverage
3. **Improve Documentation** - Fix typos, clarify instructions, add examples
4. **Report Bugs** - Open issues for bugs you encounter
5. **Suggest Features** - Share ideas for new capabilities
6. **Fix Bugs** - Submit PRs for known issues

The most impactful contribution is **adding support for new cloud providers**. This guide focuses on that workflow.

---

## Adding a New Cloud Provider

Adding a new cloud provider to Infrar requires **zero changes to platform code**. All provider-specific logic lives in the `infrar-plugins` repository under the `providers/` directory.

### Prerequisites

- Familiarity with the cloud provider's APIs and services
- Understanding of Terraform/OpenTofu
- Basic knowledge of YAML configuration
- (Optional) Knowledge of the provider's SDK for code transformation rules

### Step 1: Create Provider Directory Structure

```bash
cd infrar-plugins

# Create provider directory
mkdir -p providers/{your-provider}

# Create subdirectories
mkdir -p providers/{your-provider}/authentication
mkdir -p providers/{your-provider}/services/{compute,storage,database,messaging}
mkdir -p providers/{your-provider}/terraform-config
mkdir -p providers/{your-provider}/orchestration
```

### Step 2: Create Provider Metadata Files

#### `provider.yaml`

Define basic provider information:

```yaml
provider:
  id: digitalocean               # Unique identifier (lowercase, no spaces)
  name: DigitalOcean
  display_name: DigitalOcean
  description: "DigitalOcean cloud platform"
  version: 1.0.0

  # Capabilities this provider supports
  supported_capabilities:
    - compute
    - storage
    - database

  # Terraform provider information
  terraform_provider:
    source: "digitalocean/digitalocean"
    version: "~> 2.0"

  # Documentation
  documentation_url: "https://docs.digitalocean.com"
  console_url: "https://cloud.digitalocean.com"
```

#### `regions.yaml`

List available regions:

```yaml
regions:
  - id: nyc1
    name: "New York 1"
    location: "New York, USA"
    available: true

  - id: nyc3
    name: "New York 3"
    location: "New York, USA"
    available: true

  - id: sfo3
    name: "San Francisco 3"
    location: "San Francisco, USA"
    available: true

  - id: ams3
    name: "Amsterdam 3"
    location: "Amsterdam, Netherlands"
    available: true
```

#### `provider-defaults.yaml`

Default configuration:

```yaml
provider_defaults:
  id: digitalocean-defaults
  version: 1.0.0
  provider: digitalocean

  # Default region
  default_region: nyc3

  # Default compute settings
  compute:
    size: s-1vcpu-1gb
    image: ubuntu-22-04-x64

  # Default storage settings
  storage:
    location: nyc3

  # Label configuration
  label_format: lowercase_underscore
  label_key: tags
```

#### `pricing-metadata.yaml`

Cost estimation data:

```yaml
pricing:
  provider: digitalocean
  currency: USD
  last_updated: "2025-10-23"

  compute:
    s-1vcpu-1gb:
      vcpus: 1
      memory_gb: 1
      monthly_cost: 6.00
      hourly_cost: 0.00893

    s-2vcpu-2gb:
      vcpus: 2
      memory_gb: 2
      monthly_cost: 12.00
      hourly_cost: 0.01786

  storage:
    spaces:
      base_cost: 5.00             # $5/month for 250GB
      additional_gb_cost: 0.02     # $0.02/GB over 250GB
```

### Step 3: Add Authentication Method

Create authentication method directory:

```bash
mkdir -p providers/{your-provider}/authentication/api-token
```

#### `method.yaml`

```yaml
authentication_method:
  id: api-token
  name: "API Token"
  display_name: "DigitalOcean API Token"
  description: "Authenticate using a DigitalOcean API token"
  recommended: true

  security_level: high
  setup_complexity: easy
  rotation_supported: true
```

#### `schema.yaml`

```yaml
credential_schema:
  type: object
  required:
    - api_token
  properties:
    api_token:
      type: string
      description: "DigitalOcean API token"
      sensitive: true
      pattern: "^[a-f0-9]{64}$"
      min_length: 64
      max_length: 64
```

#### `handler.yaml`

```yaml
handler:
  setup_type: environment_variable

  environment_variables:
    DIGITALOCEAN_TOKEN:
      source: api_token
      description: "DigitalOcean API token for authentication"

  validation:
    endpoint: "https://api.digitalocean.com/v2/account"
    method: GET
    headers:
      Authorization: "Bearer ${api_token}"
    success_status: 200
```

#### `ui-instructions.yaml`

```yaml
ui_instructions:
  title: "DigitalOcean API Token Setup"
  estimated_time: "3 minutes"
  difficulty: "Easy"

  prerequisites:
    - "Active DigitalOcean account"
    - "Account with appropriate permissions"

  steps:
    - number: 1
      title: "Open DigitalOcean Control Panel"
      description: "Log in to your DigitalOcean account and navigate to API settings"
      action_type: "link"
      action_url: "https://cloud.digitalocean.com/account/api/tokens"
      action_label: "Open API Settings"

    - number: 2
      title: "Generate New Token"
      description: "Click 'Generate New Token' and give it a descriptive name like 'Infrar Platform'"
      tips:
        - "Enable both Read and Write scopes"
        - "Store the token securely - it will only be shown once"

    - number: 3
      title: "Copy Token"
      description: "Copy the generated token immediately - you won't be able to see it again"
      warning: "Store this token securely. Never commit it to version control."

    - number: 4
      title: "Enter Token"
      description: "Paste the token in the field below"
      input_field: "api_token"

  validation:
    message: "Validating token..."
    success_message: "Token validated successfully!"
    error_message: "Token validation failed. Please check the token and try again."
```

### Step 4: Add Service Capability

Let's add storage support as an example:

```bash
mkdir -p providers/{your-provider}/services/storage/spaces
```

#### `service.yaml`

```yaml
service:
  id: spaces
  name: "Spaces Object Storage"
  display_name: "DigitalOcean Spaces"
  provider: digitalocean
  category: storage
  capability_type: storage

  description: "S3-compatible object storage"
  documentation_url: "https://docs.digitalocean.com/products/spaces/"

  # Terraform resource type
  terraform_resource_type: "digitalocean_spaces_bucket"

  # Required APIs (if applicable)
  provider_apis: []

  # Supported features
  features:
    - versioning
    - cors
    - lifecycle_rules
    - cdn_integration

  # Limitations
  limitations:
    - "Maximum bucket name length: 63 characters"
    - "Bucket names must be globally unique"
```

#### `defaults.yaml`

```yaml
defaults:
  service: spaces
  provider: digitalocean

  bucket:
    acl: "private"
    force_destroy: false
    versioning: false
    region: nyc3

  cors_rules: []

  lifecycle_rules: []
```

#### `terraform/main.tf`

```hcl
# DigitalOcean Spaces Bucket
resource "digitalocean_spaces_bucket" "main" {
  name   = var.bucket_name
  region = var.region

  acl = var.acl

  # CORS configuration
  dynamic "cors_rule" {
    for_each = var.cors_rules
    content {
      allowed_headers = cors_rule.value.allowed_headers
      allowed_methods = cors_rule.value.allowed_methods
      allowed_origins = cors_rule.value.allowed_origins
      max_age_seconds = cors_rule.value.max_age_seconds
    }
  }

  # Versioning
  versioning {
    enabled = var.versioning_enabled
  }
}

# Variables
variable "bucket_name" {
  type        = string
  description = "Name of the Spaces bucket"
}

variable "region" {
  type        = string
  description = "Region for the bucket"
  default     = "nyc3"
}

variable "acl" {
  type        = string
  description = "Access control list"
  default     = "private"
}

variable "versioning_enabled" {
  type        = bool
  description = "Enable versioning"
  default     = false
}

variable "cors_rules" {
  type        = list(any)
  description = "CORS rules"
  default     = []
}

# Outputs
output "bucket_name" {
  value       = digitalocean_spaces_bucket.main.name
  description = "Name of the created bucket"
}

output "bucket_urn" {
  value       = digitalocean_spaces_bucket.main.urn
  description = "URN of the bucket"
}

output "endpoint" {
  value       = "https://${var.bucket_name}.${var.region}.digitaloceanspaces.com"
  description = "Bucket endpoint URL"
}
```

#### `transform/rules.yaml`

```yaml
transformations:
  - pattern: "infrar.storage.upload"
    target:
      provider: digitalocean
      service: spaces

    imports:
      - "import boto3"  # Spaces is S3-compatible

    template: |
      # Spaces uses S3-compatible API
      s3_client = boto3.client(
          's3',
          region_name='{{region}}',
          endpoint_url='https://{{region}}.digitaloceanspaces.com',
          aws_access_key_id=os.environ.get('SPACES_ACCESS_KEY'),
          aws_secret_access_key=os.environ.get('SPACES_SECRET_KEY')
      )
      s3_client.upload_file({{source}}, {{bucket}}, {{destination}})

    requirements:
      - package: boto3
        version: ">=1.28.0"

  - pattern: "infrar.storage.download"
    target:
      provider: digitalocean
      service: spaces

    imports:
      - "import boto3"

    template: |
      s3_client = boto3.client(
          's3',
          region_name='{{region}}',
          endpoint_url='https://{{region}}.digitaloceanspaces.com',
          aws_access_key_id=os.environ.get('SPACES_ACCESS_KEY'),
          aws_secret_access_key=os.environ.get('SPACES_SECRET_KEY')
      )
      s3_client.download_file({{bucket}}, {{source}}, {{destination}})

    requirements:
      - package: boto3
        version: ">=1.28.0"
```

### Step 5: Add Terraform Configuration Templates

#### `terraform-config/provider-block.tf.tmpl`

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}
```

#### `terraform-config/backend.tf.tmpl`

```hcl
terraform {
  backend "s3" {
    endpoint                    = "{{region}}.digitaloceanspaces.com"
    region                      = "us-east-1"  # Spaces requires this
    bucket                      = "infrar-terraform-state"
    key                         = "terraform/state/{{project_id}}/terraform.tfstate"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
  }
}
```

#### `terraform-config/variables.tf.tmpl`

```hcl
variable "do_token" {
  description = "DigitalOcean API token"
  type        = string
  sensitive   = true
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "region" {
  description = "DigitalOcean region"
  type        = string
  default     = "nyc3"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = list(string)
  default     = ["managed-by-infrar"]
}
```

### Step 6: Add Orchestration Bootstrap Script

#### `orchestration/bootstrap-script.sh.tmpl`

```bash
#!/bin/bash
set -e

echo "🚀 Bootstrapping DigitalOcean infrastructure for project: {{project_name}}"

# Verify API token
if [ -z "$DIGITALOCEAN_TOKEN" ]; then
  echo "❌ Error: DIGITALOCEAN_TOKEN environment variable not set"
  exit 1
fi

echo "✓ API token found"

# Create Spaces bucket for Terraform state (if needed)
echo "Creating Terraform state bucket..."
doctl spaces create infrar-terraform-state --region {{region}} 2>&1 || echo "  ℹ️  Bucket may already exist"

echo "✓ Terraform state bucket ready"

# Verify region availability
echo "Verifying region {{region}} availability..."
doctl compute region list --format Slug | grep -q {{region}} && echo "✓ Region available" || (echo "❌ Region not available" && exit 1)

echo ""
echo "✅ Bootstrap complete! Ready to provision infrastructure."
```

### Step 7: Test Your Provider

1. **Verify Plugin Structure**
   ```bash
   # From infrar-plugins directory
   ./verify_plugin.sh digitalocean
   ```

2. **Test Authentication**
   - Obtain test credentials from the provider
   - Test credential validation endpoint
   - Verify environment variables are set correctly

3. **Test Terraform Generation**
   - Create a test project using your provider
   - Verify terraform files are generated correctly
   - Run `terraform plan` to check for syntax errors

4. **Test Code Transformation**
   - Write sample code using infrar SDK
   - Run transformation
   - Verify generated code is correct

5. **End-to-End Test**
   - Deploy a simple storage workload
   - Verify resources are created
   - Test the deployed application
   - Clean up resources

---

## Adding a New Service to an Existing Provider

If you want to add a new service (e.g., Redis caching) to an existing provider (e.g., AWS):

1. Create service directory:
   ```bash
   mkdir -p providers/aws/services/caching/elasticache-redis
   ```

2. Add the four required files:
   - `service.yaml` - Service metadata
   - `defaults.yaml` - Default configuration
   - `terraform/main.tf` - Terraform module
   - `transform/rules.yaml` - Transformation rules

3. Update `provider.yaml` to include the new capability in `supported_capabilities`

4. Test the new service

---

## Plugin Development Guidelines

### File Naming Conventions

- Use lowercase with hyphens: `service-account-json`, `cloud-run`
- YAML files: `.yaml` (not `.yml`)
- Templates: `.tmpl` extension
- Terraform: `.tf` files

### YAML Style

```yaml
# Use 2-space indentation
key: value
nested:
  key: value

# Use double quotes for strings with special characters
description: "This is a description"

# Use lists for multiple values
items:
  - item1
  - item2
```

### Terraform Style

- Follow [Terraform style guide](https://www.terraform.io/docs/language/syntax/style.html)
- Use meaningful resource names
- Include descriptions for all variables
- Mark sensitive variables as `sensitive = true`
- Provide useful outputs

### Documentation

- All YAML files should be self-documenting with clear descriptions
- Include examples where helpful
- Reference official provider documentation
- Document any limitations or known issues

---

## Testing Your Changes

### Local Testing

1. **Plugin Structure Validation**
   ```bash
   # Validate plugin structure
   ./scripts/validate_plugin.sh {provider-name}
   ```

2. **Terraform Validation**
   ```bash
   # Generate terraform for a test project
   cd /tmp/test-project
   terraform init
   terraform validate
   terraform plan
   ```

3. **Manual Testing**
   - Create test credentials
   - Deploy a simple workload
   - Verify functionality
   - Clean up resources

### Automated Testing

Infrar includes automated tests for plugins:

```bash
# Run plugin tests
cd infrar-platform
go test ./internal/services/plugins/... -v

# Test specific provider
go test ./internal/services/plugins/... -v -run TestProvider/digitalocean
```

---

## Submitting Changes

### Before Submitting

- [ ] Plugin structure follows conventions
- [ ] All required files are present
- [ ] YAML syntax is valid
- [ ] Terraform syntax is valid
- [ ] Tested locally with real credentials
- [ ] Documentation is clear and complete
- [ ] No sensitive information (tokens, keys) committed

### Pull Request Process

1. **Fork the Repository**
   ```bash
   # Fork infrar-plugins on GitHub
   git clone https://github.com/YOUR-USERNAME/infrar-plugins.git
   cd infrar-plugins
   ```

2. **Create a Feature Branch**
   ```bash
   git checkout -b add-digitalocean-provider
   ```

3. **Make Your Changes**
   - Add all required files
   - Follow the structure outlined in this guide
   - Test thoroughly

4. **Commit Your Changes**
   ```bash
   git add providers/digitalocean
   git commit -m "Add DigitalOcean provider support

   - Add provider metadata and defaults
   - Add API token authentication
   - Add Spaces storage service
   - Add terraform templates and bootstrap script
   "
   ```

5. **Push to Your Fork**
   ```bash
   git push origin add-digitalocean-provider
   ```

6. **Create Pull Request**
   - Go to the original infrar-plugins repository
   - Click "New Pull Request"
   - Select your fork and branch
   - Fill in the PR template:
     - **Description**: What provider/service you're adding
     - **Testing**: How you tested it
     - **Checklist**: Complete the checklist

7. **Address Review Feedback**
   - Maintainers will review your PR
   - Address any requested changes
   - Push updates to your branch

8. **Merge**
   - Once approved, maintainers will merge your PR
   - Your provider will be available in the next release!

---

## Need Help?

- **Documentation**: https://docs.infrar.io
- **Plugin Examples**: Check existing providers in `providers/`
- **Issues**: https://github.com/QodeSrl/infrar-plugins/issues
- **Discussions**: https://github.com/QodeSrl/infrar-plugins/discussions

Thank you for contributing to Infrar! 🎉
