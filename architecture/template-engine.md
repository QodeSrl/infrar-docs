# Template Engine

## Overview

The **Template Engine** is a powerful, centralized system for rendering Terraform configurations, bootstrap scripts, and other provider-specific resources from data-driven templates. It eliminates hardcoded terraform generation logic and enables all provider-specific code generation to be template-based.

## Motivation

### Problems Solved

1. **Hardcoded Terraform Generation**
   - **Before**: Provider blocks, backends, and resources generated with string concatenation in Go
   - **After**: All terraform code generated from templates in plugin directories

2. **Inflexible Code Generation**
   - **Before**: Adding new provider-specific features required platform code changes
   - **After**: Update templates without touching platform code

3. **Repetitive Rendering Logic**
   - **Before**: Multiple places with duplicate template rendering code
   - **After**: Single, centralized template engine with rich helper functions

4. **Limited Template Capabilities**
   - **Before**: Basic Go template functionality only
   - **After**: 50+ helper functions (sanitize, json, yaml, terraform-specific, etc.)

### Benefits

- ✅ **Centralized**: One template engine for all rendering needs
- ✅ **Powerful**: 50+ built-in helper functions
- ✅ **Type-Safe**: Structured context with Go types
- ✅ **Extensible**: Easy to add new helper functions
- ✅ **Testable**: Comprehensive test suite
- ✅ **Modular**: Templates live in plugin directories

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    Template Engine                         │
│                                                            │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Context Builder                                 │    │
│  │  - Provider info                                 │    │
│  │  - Project info                                  │    │
│  │  - Capabilities                                  │    │
│  │  - Variables                                     │    │
│  │  - Metadata                                      │    │
│  └────────────┬─────────────────────────────────────┘    │
│               │                                            │
│  ┌────────────▼─────────────────────────────────────┐    │
│  │  Template Renderer                               │    │
│  │  - Load templates from files or strings         │    │
│  │  - Cache compiled templates                     │    │
│  │  - Execute with context                         │    │
│  └────────────┬─────────────────────────────────────┘    │
│               │                                            │
│  ┌────────────▼─────────────────────────────────────┐    │
│  │  Function Registry (50+ helpers)                │    │
│  │  - String manipulation (upper, lower, etc.)     │    │
│  │  - Sanitization (sanitize, toSnakeCase, etc.)   │    │
│  │  - Encoding (json, yaml, base64)                │    │
│  │  - Terraform-specific (tfvar, tfstring, etc.)   │    │
│  │  - Cloud providers (gcpLabel, awsTag, etc.)     │    │
│  │  - Utilities (indent, quote, hash, etc.)        │    │
│  └──────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────┐
        │       Generated Output            │
        │  - main.tf (provider block)       │
        │  - backend.tf (state config)      │
        │  - variables.tf (var definitions) │
        │  - bootstrap.sh (setup script)    │
        └───────────────────────────────────┘
```

---

## Core Components

### 1. Engine

The main template engine that orchestrates rendering.

```go
type Engine struct {
    basePath  string                          // Base path for template files
    templates map[string]*template.Template   // Cached templates
    funcMap   template.FuncMap                // Helper functions
}
```

**Methods**:
- `RenderString(templateStr, ctx)` - Render a template string
- `RenderFile(path, ctx)` - Render a template file
- `LoadTemplate(name, path)` - Load and cache a template
- `Render(name, ctx)` - Render a cached template

### 2. Context

The data structure passed to templates containing all available variables.

```go
type Context struct {
    // Provider information
    Provider   string
    ProviderID string
    Region     string

    // Project information
    ProjectID   string
    ProjectName string
    Environment string

    // Capabilities (detected from user code)
    Capabilities []Capability

    // Variables (user-provided and computed)
    Variables    map[string]interface{}

    // Metadata (extracted from credentials)
    Metadata     map[string]interface{}

    // Workspace information
    Workspace    string

    // Custom data (extensible)
    Custom       map[string]interface{}
}
```

### 3. Function Registry

50+ template helper functions organized by category.

---

## Helper Functions

### String Manipulation

| Function | Description | Example |
|----------|-------------|---------|
| `upper` | Convert to uppercase | `{{ "hello" \| upper }}` → `HELLO` |
| `lower` | Convert to lowercase | `{{ "HELLO" \| lower }}` → `hello` |
| `title` | Title case | `{{ "hello world" \| title }}` → `Hello World` |
| `trim` | Trim whitespace | `{{ "  hello  " \| trim }}` → `hello` |
| `trimPrefix` | Remove prefix | `{{ "hello-world" \| trimPrefix "hello-" }}` → `world` |
| `trimSuffix` | Remove suffix | `{{ "hello-world" \| trimSuffix "-world" }}` → `hello` |
| `replace` | Replace all | `{{ "hello world" \| replace " " "-" }}` → `hello-world` |
| `split` | Split string | `{{ "a,b,c" \| split "," }}` → `["a", "b", "c"]` |
| `join` | Join strings | `{{ list "a" "b" "c" \| join "," }}` → `a,b,c` |
| `contains` | Check substring | `{{ "hello" \| contains "ell" }}` → `true` |
| `hasPrefix` | Check prefix | `{{ "hello" \| hasPrefix "he" }}` → `true` |
| `hasSuffix` | Check suffix | `{{ "hello" \| hasSuffix "lo" }}` → `true` |

### Sanitization

| Function | Description | Example |
|----------|-------------|---------|
| `sanitize` | Clean resource name | `{{ "My Project" \| sanitize }}` → `my-project` |
| `sanitizeLabel` | Clean label (GCP) | `{{ "My Label" \| sanitizeLabel }}` → `my-label` |
| `sanitizeDNS` | Clean DNS name | `{{ "my_service" \| sanitizeDNS }}` → `my-service` |
| `toSnakeCase` | snake_case | `{{ "MyProject" \| toSnakeCase }}` → `my_project` |
| `toCamelCase` | camelCase | `{{ "my-project" \| toCamelCase }}` → `myProject` |
| `toKebabCase` | kebab-case | `{{ "MyProject" \| toKebabCase }}` → `my-project` |

### Encoding

| Function | Description | Example |
|----------|-------------|---------|
| `json` | Convert to JSON | `{{ .Variables.tags \| json }}` |
| `jsonPretty` | Pretty JSON | `{{ .Variables.tags \| jsonPretty }}` |
| `yaml` | Convert to YAML | `{{ .Variables.tags \| yaml }}` |
| `base64` | Base64 encode | `{{ "hello" \| base64 }}` → `aGVsbG8=` |
| `base64Decode` | Base64 decode | `{{ "aGVsbG8=" \| base64Decode }}` → `hello` |

### Hashing

| Function | Description | Example |
|----------|-------------|---------|
| `sha256` | SHA256 hash | `{{ "password" \| sha256 }}` |
| `md5` | MD5 hash | `{{ "password" \| md5 }}` |

### Lists and Maps

| Function | Description | Example |
|----------|-------------|---------|
| `list` | Create list | `{{ list "a" "b" "c" }}` |
| `dict` | Create map | `{{ dict "key1" "val1" "key2" "val2" }}` |
| `get` | Get map value | `{{ .Variables \| get "region" }}` |
| `set` | Set map value | `{{ .Variables \| set "key" "value" }}` |
| `has` | Check key exists | `{{ .Variables \| has "region" }}` |
| `keys` | Get map keys | `{{ .Variables \| keys }}` |
| `values` | Get map values | `{{ .Variables \| values }}` |

### Conditionals

| Function | Description | Example |
|----------|-------------|---------|
| `default` | Default value | `{{ .Variables.region \| default "us-east-1" }}` |
| `empty` | Check if empty | `{{ if empty .Variables.zone }}...{{ end }}` |
| `notEmpty` | Check not empty | `{{ if notEmpty .Variables.region }}...{{ end }}` |
| `coalesce` | First non-empty | `{{ coalesce .Var1 .Var2 "default" }}` |

### Numeric

| Function | Description | Example |
|----------|-------------|---------|
| `add` | Addition | `{{ add 5 3 }}` → `8` |
| `sub` | Subtraction | `{{ sub 5 3 }}` → `2` |
| `mul` | Multiplication | `{{ mul 5 3 }}` → `15` |
| `div` | Division | `{{ div 6 3 }}` → `2` |
| `mod` | Modulo | `{{ mod 7 3 }}` → `1` |

### Date/Time

| Function | Description | Example |
|----------|-------------|---------|
| `now` | Current time | `{{ now }}` |
| `unixTime` | Unix timestamp | `{{ unixTime }}` → `1730454123` |
| `formatDate` | Format date | `{{ formatDate "2006-01-02" now }}` |

### Terraform-Specific

| Function | Description | Example |
|----------|-------------|---------|
| `tfvar` | Variable reference | `{{ tfvar "project_id" }}` → `var.project_id` |
| `tfstring` | Terraform string | `{{ tfstring "my-project" }}` → `"my-project"` |
| `tfbool` | Terraform boolean | `{{ tfbool true }}` → `true` |
| `tfnumber` | Terraform number | `{{ tfnumber 42 }}` → `42` |
| `tflist` | Terraform list | `{{ tflist (list "a" "b") }}` → `["a", "b"]` |
| `tfmap` | Terraform map | `{{ tfmap .Variables.tags }}` → `{key = "value"}` |

### Cloud Provider-Specific

| Function | Description | Example |
|----------|-------------|---------|
| `gcpLabel` | GCP label format | `{{ gcpLabel "env" "prod" }}` → `env = "prod"` |
| `awsTag` | AWS tag format | `{{ awsTag "Env" "Prod" }}` → `"Env" = "Prod"` |
| `azureTag` | Azure tag format | `{{ azureTag "Env" "Prod" }}` → `"Env" = "Prod"` |

### Utilities

| Function | Description | Example |
|----------|-------------|---------|
| `indent` | Indent lines | `{{ "text" \| indent 4 }}` |
| `nindent` | Newline + indent | `{{ "text" \| nindent 4 }}` |
| `quote` | Double quote | `{{ "text" \| quote }}` → `"text"` |
| `squote` | Single quote | `{{ "text" \| squote }}` → `'text'` |
| `repeat` | Repeat string | `{{ repeat "*" 5 }}` → `*****` |
| `concat` | Concatenate | `{{ concat "hello" "-" "world" }}` → `hello-world` |

---

## Usage Examples

### Example 1: GCP Provider Block

**Template** (`providers/gcp/terraform-config/provider-block.tf.tmpl`):

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
  region  = {{ .Variables.region | default "us-central1" | tfstring }}

  {{- if .Variables.zone }}
  zone    = {{ .Variables.zone | tfstring }}
  {{- end }}
}
```

**Go Code**:

```go
engine := template.NewEngine("/path/to/infrar-plugins")

ctx := &template.Context{
    Provider: "gcp",
    Variables: map[string]interface{}{
        "region": "us-west1",
    },
}

output, err := engine.RenderFile("providers/gcp/terraform-config/provider-block.tf.tmpl", ctx)
```

**Output**:

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
  region  = "us-west1"
}
```

### Example 2: Capability-Based Bootstrap Script

**Template** (`providers/gcp/orchestration/bootstrap-script.sh.tmpl`):

```bash
#!/bin/bash
set -e

PROJECT_ID="{{ .Metadata.project_id }}"

{{- if .Capabilities }}
{{- range .Capabilities }}
{{- if eq .Type "storage" }}
gcloud services enable storage.googleapis.com --project=$PROJECT_ID
{{- end }}
{{- if eq .Type "compute" }}
gcloud services enable run.googleapis.com --project=$PROJECT_ID
{{- end }}
{{- end }}
{{- end }}

echo "✓ Bootstrap complete"
```

**Go Code**:

```go
ctx := &template.Context{
    Metadata: map[string]interface{}{
        "project_id": "my-gcp-project",
    },
    Capabilities: []template.Capability{
        {Type: "storage"},
        {Type: "compute"},
    },
}

output, err := engine.RenderFile("providers/gcp/orchestration/bootstrap-script.sh.tmpl", ctx)
```

**Output**:

```bash
#!/bin/bash
set -e

PROJECT_ID="my-gcp-project"
gcloud services enable storage.googleapis.com --project=$PROJECT_ID
gcloud services enable run.googleapis.com --project=$PROJECT_ID

echo "✓ Bootstrap complete"
```

### Example 3: Dynamic Variables with Capabilities

**Template** (`providers/gcp/terraform-config/variables.tf.tmpl`):

```hcl
variable "project_id" {
  description = "GCP project ID"
  type        = string
  {{- if .Metadata.project_id }}
  default     = {{ .Metadata.project_id | tfstring }}
  {{- end }}
}

{{- range .Capabilities }}
{{- if eq .Type "compute" }}

variable "cpu_limit" {
  description = "CPU limit"
  type        = string
  default     = {{ .Parameters.cpu_limit | default "1000m" | tfstring }}
}
{{- end }}
{{- end }}
```

**Context**:

```go
ctx := &template.Context{
    Metadata: map[string]interface{}{
        "project_id": "extracted-from-json",
    },
    Capabilities: []template.Capability{
        {
            Type: "compute",
            Parameters: map[string]interface{}{
                "cpu_limit": "2000m",
            },
        },
    },
}
```

**Output**:

```hcl
variable "project_id" {
  description = "GCP project ID"
  type        = string
  default     = "extracted-from-json"
}

variable "cpu_limit" {
  description = "CPU limit"
  type        = string
  default     = "2000m"
}
```

---

## Integration with Platform

### Where Template Engine is Used

1. **Terraform Generation** (`terraform_generator.go`)
   - Generate `main.tf` (provider block)
   - Generate `backend.tf` (state configuration)
   - Generate `variables.tf` (variable definitions)
   - Generate capability-specific resources

2. **Bootstrap Scripts** (`orchestrator.go`)
   - Generate provider-specific API enablement scripts
   - Generate setup scripts

3. **Cross-Capability IAM** (`iam_generator.go`)
   - Generate IAM bindings based on detected capabilities
   - Provider-specific permission templates

4. **Credential Handling** (`credential_interpreter.go`)
   - Already using basic template rendering
   - Can be upgraded to use centralized engine

### Migration from Old System

**Before** (Hardcoded):

```go
func (tg *TerraformGenerator) generateProviderBlock(provider string) (string, error) {
    var buf bytes.Buffer
    buf.WriteString("terraform {\n")
    buf.WriteString("  required_version = \">= 1.0\"\n")

    switch provider {
    case "gcp":
        buf.WriteString("    google = {\n")
        buf.WriteString("      source  = \"hashicorp/google\"\n")
        // ... 20+ more lines
    }

    return buf.String(), nil
}
```

**After** (Template-Based):

```go
func (tg *TerraformGenerator) generateProviderBlock(provider string, ctx *template.Context) (string, error) {
    templatePath := fmt.Sprintf("providers/%s/terraform-config/provider-block.tf.tmpl", provider)
    return tg.engine.RenderFile(templatePath, ctx)
}
```

**Result**: 30+ lines of hardcoded Go → 3 lines + template file

---

## Testing

Run the comprehensive test suite:

```bash
cd /home/alexb/projects/infrar/infrar-platform
go run test_template_engine.go
```

**Tests Covered**:

1. ✅ String manipulation functions (upper, lower, sanitize, etc.)
2. ✅ Terraform-specific functions (tfvar, tfstring, tfmap, etc.)
3. ✅ Cloud provider label formatting (gcpLabel, awsTag, etc.)
4. ✅ Conditionals and defaults (default, empty, coalesce)
5. ✅ JSON and YAML encoding
6. ✅ Rendering GCP provider block from template
7. ✅ Rendering GCP backend config from template
8. ✅ Rendering GCP variables from template
9. ✅ Rendering bootstrap script from template
10. ✅ Capability-based rendering
11. ✅ Utility functions (indent, quote, hash, base64)

**Result**: All 11 tests passing ✓

---

## Performance

- **Template Parsing**: One-time cost per template (~1-2ms)
- **Template Caching**: Compiled templates cached in memory
- **Rendering**: <1ms per template execution
- **Memory**: ~100KB per cached template

**Best Practice**: Use `LoadTemplate()` + `Render()` for frequently-used templates instead of `RenderFile()` repeatedly.

---

## Future Enhancements

### Planned Features

1. **Template Validation**
   - Syntax validation before rendering
   - Type checking for context fields
   - Missing variable detection

2. **Template Composition**
   - Include/import other templates
   - Template inheritance
   - Partial templates

3. **Additional Helper Functions**
   - Regular expression matching
   - Advanced date manipulation
   - More cloud provider-specific formatters

4. **Template Debugging**
   - Verbose logging mode
   - Template execution tracing
   - Variable inspection

5. **Multi-Language Support**
   - Support for other template engines (Jinja2, etc.)
   - Language-specific helpers

---

## Best Practices

### 1. Use Meaningful Variable Names

```hcl
# ❌ Bad
{{ .V.r }}

# ✅ Good
{{ .Variables.region }}
```

### 2. Provide Defaults for Optional Values

```hcl
# ✅ Always provide defaults
region = {{ .Variables.region | default "us-central1" | tfstring }}
```

### 3. Use Conditionals to Avoid Empty Blocks

```hcl
{{- if .Variables.zone }}
zone = {{ .Variables.zone | tfstring }}
{{- end }}
```

### 4. Sanitize User-Provided Names

```hcl
# ✅ Sanitize before using
bucket_name = {{ .ProjectName | sanitize | tfstring }}
```

### 5. Use Provider-Specific Label Functions

```hcl
# ✅ Use gcpLabel for GCP
labels = {
  {{ range $key, $value := .Variables.tags }}
  {{ gcpLabel $key $value }}
  {{ end }}
}
```

---

## See Also

- [Authentication Plugins](authentication-plugins.md) - Uses template engine for credential setup
- [Provider Plugins](provider-plugins.md) - Template structure in plugins
- [Terraform Generation](terraform-generation.md) - How templates generate terraform code

---

## Changelog

- **2025-10-23**: Initial implementation with 50+ helper functions
- **2025-10-23**: Comprehensive test suite (11 tests, all passing)
- **2025-10-23**: GCP provider block, backend, variables, and bootstrap templates created
