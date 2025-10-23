# Authentication Plugin System

## Overview

The Authentication Plugin System enables **provider-agnostic credential management** in Infrar. Instead of hardcoding authentication logic for each cloud provider in the platform code, all authentication methods are defined as **data-driven plugins** using YAML configuration files.

## Why Authentication Plugins?

### Problems Solved

1. **Hardcoded Credential Logic**: Previously, credential instructions and handling were hardcoded in `infrar-platform/internal/api/deployment/handlers.go` (160+ lines per provider)
2. **Inflexibility**: Adding a new authentication method required platform code changes
3. **Multiple Methods**: Cloud providers offer multiple authentication methods (service accounts, workload identity, OAuth, etc.) - all need to be supported
4. **UI Coupling**: UI instructions were embedded in Go code, making them hard to update and translate

### Benefits

- ✅ **Modular**: Each authentication method is self-contained
- ✅ **Extensible**: Add new methods without changing platform code
- ✅ **Data-Driven**: All configuration in YAML files
- ✅ **Discoverable**: Methods are automatically loaded and exposed via API
- ✅ **Multi-Method**: Support multiple authentication methods per provider
- ✅ **Translatable**: UI instructions in separate files for easy localization

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  infrar-platform                         │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │  AuthenticationHandler (API Layer)             │    │
│  │  GET /providers/:provider/auth-methods         │    │
│  │  GET /providers/:provider/auth-methods/:id     │    │
│  └────────────────┬───────────────────────────────┘    │
│                   │                                      │
│  ┌────────────────▼───────────────────────────────┐    │
│  │  ProviderRegistry (Business Logic)             │    │
│  │  - LoadProviders()                             │    │
│  │  - GetAuthenticationMethods(provider)          │    │
│  │  - GetAuthenticationMethod(provider, methodID) │    │
│  └────────────────┬───────────────────────────────┘    │
│                   │                                      │
│  ┌────────────────▼───────────────────────────────┐    │
│  │  AuthMethodLoader (Loader)                     │    │
│  │  - LoadProviderAuthMethods(provider)           │    │
│  │  - LoadAuthMethod(provider, method)            │    │
│  └────────────────┬───────────────────────────────┘    │
└───────────────────┼──────────────────────────────────────┘
                    │ Loads from
                    ▼
┌──────────────────────────────────────────────────────────┐
│              infrar-plugins (Data Layer)                 │
│                                                          │
│  providers/                                              │
│  ├── gcp/                                                │
│  │   └── authentication/                                │
│  │       ├── service-account-json/                      │
│  │       │   ├── method.yaml          # Metadata        │
│  │       │   ├── schema.yaml          # Field defs      │
│  │       │   ├── handler.yaml         # Setup logic     │
│  │       │   └── ui-instructions.yaml # UI steps        │
│  │       │                                               │
│  │       ├── workload-identity/       # Future          │
│  │       └── oauth/                   # Future          │
│  │                                                       │
│  ├── aws/                                                │
│  │   └── authentication/                                │
│  │       ├── access-keys/                               │
│  │       ├── iam-role/                                  │
│  │       └── sso/                                       │
│  │                                                       │
│  └── azure/                                              │
│      └── authentication/                                │
│          ├── service-principal/                         │
│          ├── managed-identity/                          │
│          └── certificate/                               │
└──────────────────────────────────────────────────────────┘
```

## Plugin Structure

Each authentication method is defined in its own directory with 4 YAML files:

### Directory Layout

```
providers/{provider}/authentication/{method-name}/
├── method.yaml              # Metadata about the authentication method
├── schema.yaml              # Credential fields and validation rules
├── handler.yaml             # How to setup credentials for terraform
└── ui-instructions.yaml     # Step-by-step UI instructions for users
```

### File Purposes

#### 1. `method.yaml` - Authentication Method Metadata

Defines high-level information about the authentication method:

- Identity (ID, name, display name)
- Security characteristics (credential type, rotation requirements)
- Supported environments (local dev, CI/CD, production)
- Prerequisites and required permissions
- Alternative methods and when to use them
- Documentation links

**Example**: [providers/gcp/authentication/service-account-json/method.yaml](../../infrar-plugins/providers/gcp/authentication/service-account-json/method.yaml)

#### 2. `schema.yaml` - Credential Schema

Defines the structure and validation rules for credentials:

- Field definitions (name, type, required, sensitive)
- Validation rules (file size, format, JSON structure)
- Metadata extraction (e.g., extract project_id from JSON)
- Security warnings and best practices

**Example**: [providers/gcp/authentication/service-account-json/schema.yaml](../../infrar-plugins/providers/gcp/authentication/service-account-json/schema.yaml)

#### 3. `handler.yaml` - Credential Handler

Defines how to setup credentials for terraform execution:

- Files to create in workspace
- Environment variables to set
- Validation steps
- Troubleshooting common errors
- Cleanup procedures

**Example**: [providers/gcp/authentication/service-account-json/handler.yaml](../../infrar-plugins/providers/gcp/authentication/service-account-json/handler.yaml)

#### 4. `ui-instructions.yaml` - UI Instructions

Defines user-facing setup instructions:

- Step-by-step walkthrough
- Action buttons (links to cloud console)
- Screenshots and visual aids
- Dynamic content (show only relevant permissions)
- Warnings and security notes
- FAQ and troubleshooting
- Form field configuration

**Example**: [providers/gcp/authentication/service-account-json/ui-instructions.yaml](../../infrar-plugins/providers/gcp/authentication/service-account-json/ui-instructions.yaml)

## How It Works

### 1. Loading Plugins

On platform startup, `ProviderRegistry` loads all authentication methods:

```go
registry := plugins.NewProviderRegistry("/path/to/infrar-plugins")
registry.LoadProviders()
// [INFO] Loaded 1 authentication method(s) for provider gcp
```

### 2. Discovering Methods

Frontend queries available authentication methods:

```http
GET /api/v1/providers/gcp/auth-methods
```

Response:
```json
{
  "provider": "gcp",
  "methods": [
    {
      "id": "service-account-json",
      "display_name": "Service Account JSON Key",
      "description": "Authenticate using a GCP service account JSON key file",
      "recommended": true,
      "complexity": "simple",
      "maturity": "stable"
    }
  ],
  "total": 1
}
```

### 3. Getting UI Instructions

Frontend fetches setup instructions:

```http
GET /api/v1/providers/gcp/auth-methods/service-account-json/ui-instructions
```

Response contains complete UI rendering data:
- Step-by-step instructions
- Action buttons with URLs
- Form field definitions
- Validation messages
- FAQ entries

### 4. Credential Submission

User uploads credentials:

```http
POST /api/v1/projects/:id/credentials
{
  "method_id": "service-account-json",
  "gcp_service_account_json": "{ ... service account JSON ... }"
}
```

### 5. Credential Setup

During terraform execution, platform:
1. Retrieves authentication method handler
2. Creates required files (e.g., `gcp-service-account.json`)
3. Sets environment variables (e.g., `GOOGLE_APPLICATION_CREDENTIALS`)
4. Extracts metadata (e.g., `project_id` from JSON)

## Code Migration

### Before (Hardcoded)

```go
// handlers.go (deployment package) - 160+ lines per provider

func (h *Handler) GetCredentialInstructions(c *gin.Context) {
    var instructions gin.H

    switch provider {
    case "gcp":
        instructions = gin.H{
            "provider": "gcp",
            "title": "GCP Credentials Setup",
            "steps": []gin.H{
                {
                    "number": 1,
                    "title": "Open GCP Console",
                    "description": "Go to Google Cloud Console",
                    "link": "https://console.cloud.google.com",
                },
                // ... 160+ more lines of hardcoded JSON
            },
        }
    case "aws":
        // Another 160+ lines
    }

    c.JSON(http.StatusOK, instructions)
}
```

### After (Plugin-Based)

```go
// authentication/handlers.go - Provider-agnostic

func (h *Handler) GetAuthMethodUIInstructions(c *gin.Context) {
    provider := c.Param("provider")
    methodID := c.Param("method_id")

    method, err := h.registry.GetAuthenticationMethod(provider, methodID)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "provider":  provider,
        "method_id": methodID,
        "ui":        method.UIInstructions, // Loaded from YAML!
    })
}
```

**Result**: Platform code reduced from **480+ lines** (160 per provider × 3 providers) to **20 lines** (provider-agnostic)

## Adding a New Authentication Method

### Example: Add GCP Workload Identity

1. **Create directory**:
   ```bash
   mkdir -p infrar-plugins/providers/gcp/authentication/workload-identity
   ```

2. **Create `method.yaml`**:
   ```yaml
   authentication_method:
     id: workload-identity
     name: workload-identity
     display_name: Workload Identity
     provider: gcp
     description: Use Workload Identity for GKE/Cloud Run workloads
     recommended: false  # Not for external deployments
     complexity: advanced
     maturity: stable
     security:
       credential_type: temporary
       rotation_required: false  # Automatic rotation
       risk_level: low
     supported_environments:
       - gke
       - cloud_run
     use_cases:
       - Workloads running on GKE
       - Cloud Run services
   ```

3. **Create `schema.yaml`**:
   ```yaml
   credential_schema:
     id: gcp-workload-identity
     version: 1.0.0
     fields:
       - name: service_account_email
         type: text
         required: true
         label: Service Account Email
         description: Email of the service account to impersonate
   ```

4. **Create `handler.yaml`**:
   ```yaml
   credential_handler:
     id: gcp-workload-identity-handler
     version: 1.0.0
     method_id: workload-identity
     environment_variables:
       - name: GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
         value_template: "{{ .service_account_email }}"
         required: true
   ```

5. **Create `ui-instructions.yaml`**:
   ```yaml
   ui_instructions:
     title: "GCP Workload Identity Setup"
     steps:
       - number: 1
         title: "Enable Workload Identity"
         description: "Configure your GKE cluster"
         # ... more steps
   ```

6. **No platform code changes needed!** The new method is automatically:
   - Loaded by `ProviderRegistry`
   - Exposed via API
   - Available in the UI

## Migration Status

### ✅ Completed

- [x] Authentication plugin structure defined
- [x] GCP service-account-json method implemented
- [x] Plugin loader implemented
- [x] Registry extended for authentication methods
- [x] API handlers created
- [x] System tested end-to-end

### 🚧 In Progress

- [ ] Migrate AWS authentication methods
- [ ] Migrate Azure authentication methods
- [ ] Update frontend to use new API endpoints

### 📋 Future Work

- [ ] Add GCP Workload Identity method
- [ ] Add AWS IAM Role method
- [ ] Add Azure Managed Identity method
- [ ] Implement credential validation API
- [ ] Add i18n support for UI instructions

## API Reference

### List Authentication Methods

```http
GET /api/v1/providers/:provider/auth-methods
```

Returns all authentication methods for a provider.

**Response**:
```json
{
  "provider": "gcp",
  "methods": [
    {
      "id": "service-account-json",
      "name": "service-account-json",
      "display_name": "Service Account JSON Key",
      "description": "Authenticate using a GCP service account JSON key file",
      "recommended": true,
      "complexity": "simple",
      "maturity": "stable",
      "security": {
        "credential_type": "long-lived",
        "rotation_required": true,
        "rotation_period_days": 90
      },
      "use_cases": [
        "Local development and testing",
        "CI/CD pipelines"
      ]
    }
  ],
  "total": 1
}
```

### Get Authentication Method Details

```http
GET /api/v1/providers/:provider/auth-methods/:method_id
```

Returns complete details including metadata and schema.

### Get UI Instructions

```http
GET /api/v1/providers/:provider/auth-methods/:method_id/ui-instructions
```

Returns step-by-step UI instructions for credential setup.

### Get Recommended Method

```http
GET /api/v1/providers/:provider/auth-methods/recommended
```

Returns the recommended authentication method for a provider.

## Testing

Run the test suite:

```bash
cd infrar-platform
go run test_auth_plugin.go
```

Expected output:
```
========================================
Authentication Plugin System Test
========================================

✓ Loaded 2 provider(s): [gcp aws]
✓ Found 1 authentication method(s) for GCP
✓ Method Metadata: service-account-json
✓ Schema: 1 field(s), 3 metadata extraction rule(s)
✓ Handler: 1 file(s), 3 environment variable(s)
✓ UI Instructions: 7 step(s), 4 FAQ entries

========================================
✓ All Tests Passed!
========================================
```

## Design Principles

1. **Data Over Code**: All authentication logic defined in YAML, not Go
2. **Self-Describing**: Each method contains all metadata needed for UI/validation
3. **Discoverable**: Methods are automatically found and loaded
4. **Extensible**: Add new methods without platform changes
5. **Versioned**: Each method, schema, and handler has version numbers
6. **Backward Compatible**: Legacy `credentials.handler.yaml` still supported

## See Also

- [Provider Plugin System](provider-plugins.md)
- [Plugin Development Guide](../development/plugin-development.md)
- [API Reference](../api/authentication.md)
