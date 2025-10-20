# Security Architecture

## Overview

Security is paramount for Infrar, as the platform manages customer cloud credentials and deploys infrastructure on their behalf. This document outlines the security architecture, threat model, and mitigation strategies.

## Threat Model

### Assets to Protect

1. **Customer Cloud Credentials**: AWS keys, GCP service accounts, Azure credentials
2. **Customer Source Code**: Application code from GitHub repositories
3. **Infrastructure State**: OpenTofu state files with sensitive data
4. **User Data**: Email addresses, project configurations, API keys
5. **Infrar Platform**: API, database, build workers

### Threat Actors

1. **External Attackers**: Attempting to breach platform
2. **Compromised Accounts**: Stolen user credentials
3. **Malicious Insiders**: Rogue employees (mitigated by access controls)
4. **Supply Chain Attacks**: Compromised dependencies

### Attack Vectors

1. API vulnerabilities (injection, authentication bypass)
2. Credential theft (database breach, memory dump)
3. Code injection (malicious transformations)
4. Man-in-the-middle (network interception)
5. Social engineering (phishing users)

## Security Principles

### 1. Defense in Depth

Multiple layers of security - if one fails, others protect:

```
┌─────────────────────────────────────────────┐
│ Layer 7: Audit & Monitoring                │
├─────────────────────────────────────────────┤
│ Layer 6: Data Protection (Encryption)      │
├─────────────────────────────────────────────┤
│ Layer 5: Authorization (RBAC)              │
├─────────────────────────────────────────────┤
│ Layer 4: Authentication (OAuth 2.0)        │
├─────────────────────────────────────────────┤
│ Layer 3: Application Security (Input Val)  │
├─────────────────────────────────────────────┤
│ Layer 2: Network Security (TLS, Firewall)  │
├─────────────────────────────────────────────┤
│ Layer 1: Infrastructure Security (Hardened)│
└─────────────────────────────────────────────┘
```

### 2. Least Privilege

Every component has only the permissions it needs:
- Users can only access their own projects
- API services have minimal IAM permissions
- Customer cloud credentials scoped to required resources only

### 3. Zero Trust

Never trust, always verify:
- All API requests authenticated
- All data access authorized
- All actions audited

### 4. Encryption Everywhere

- Data at rest: Encrypted (AES-256)
- Data in transit: TLS 1.3
- Secrets: Vault with encryption

## Authentication & Authorization

### User Authentication

**OAuth 2.0 with GitHub**:
```
1. User clicks "Sign in with GitHub"
   ↓
2. Redirect to GitHub OAuth
   ↓
3. User authorizes Infrar
   ↓
4. GitHub returns authorization code
   ↓
5. Infrar exchanges code for access token
   ↓
6. Infrar creates user session (JWT)
   ↓
7. User authenticated
```

**JWT Token Structure**:
```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "iat": 1704067200,
  "exp": 1704153600,
  "iss": "infrar.io",
  "aud": "infrar-api"
}
```

**Token Security**:
- Short-lived (24 hours)
- Signed with RSA-256
- Stored in httpOnly cookie (XSS protection)
- CSRF token required for state-changing operations

### Authorization

**Role-Based Access Control (RBAC)**:

```
Roles:
├── Admin: Full access to all projects
├── Developer: Read/write to assigned projects
└── Viewer: Read-only access to assigned projects

Permissions:
├── project:read
├── project:write
├── project:delete
├── deploy:trigger
├── migrate:trigger
└── credentials:manage
```

**Middleware**:
```go
func RequireAuth(requiredPermission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Extract JWT from cookie
        token := c.Cookie("auth_token")

        // Validate JWT
        claims, err := validateJWT(token)
        if err != nil {
            c.JSON(401, gin.H{"error": "Unauthorized"})
            c.Abort()
            return
        }

        // Check permission
        if !hasPermission(claims.UserID, requiredPermission) {
            c.JSON(403, gin.H{"error": "Forbidden"})
            c.Abort()
            return
        }

        c.Set("user_id", claims.UserID)
        c.Next()
    }
}

// Usage
router.DELETE("/api/projects/:id", RequireAuth("project:delete"), deleteProject)
```

## Credential Management

### Storage (MVP Approach)

**PostgreSQL with Encryption**:
```sql
CREATE TABLE provider_credentials (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    provider VARCHAR(50) NOT NULL,
    encrypted_credentials TEXT NOT NULL,  -- AES-256 encrypted
    encryption_key_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP,
    last_used_at TIMESTAMP
);
```

**Encryption Process**:
```go
import "crypto/aes"
import "crypto/cipher"
import "crypto/rand"

func EncryptCredentials(credentials []byte, masterKey []byte) ([]byte, error) {
    // Generate unique encryption key (DEK - Data Encryption Key)
    dek := make([]byte, 32) // 256 bits
    rand.Read(dek)

    // Encrypt credentials with DEK
    block, _ := aes.NewCipher(dek)
    gcm, _ := cipher.NewGCM(block)
    nonce := make([]byte, gcm.NonceSize())
    rand.Read(nonce)

    encryptedCreds := gcm.Seal(nonce, nonce, credentials, nil)

    // Encrypt DEK with master key (KEK - Key Encryption Key)
    encryptedDEK := encryptWithMasterKey(dek, masterKey)

    // Store both
    return combine(encryptedCreds, encryptedDEK), nil
}
```

**Key Management**:
- Master key stored in environment variable (MVP)
- Production: Use AWS KMS, GCP Cloud KMS, or HashiCorp Vault
- Keys rotated every 90 days

### Production Approach: HashiCorp Vault

```go
import "github.com/hashicorp/vault/api"

func StoreCredentials(userID, provider string, creds []byte) error {
    client, _ := api.NewClient(&api.Config{
        Address: "https://vault.infrar.io",
    })

    // Authenticate with Vault
    client.SetToken(os.Getenv("VAULT_TOKEN"))

    // Store credentials
    path := fmt.Sprintf("secret/credentials/%s/%s", userID, provider)
    _, err := client.Logical().Write(path, map[string]interface{}{
        "credentials": base64.Encode(creds),
        "provider":    provider,
        "created_at":  time.Now().Unix(),
    })

    return err
}

func RetrieveCredentials(userID, provider string) ([]byte, error) {
    // Similar read from Vault
}
```

**Benefits**:
- Centralized secret management
- Automatic encryption
- Audit logs (who accessed what, when)
- Secret rotation
- Access policies

### Credential Scoping

Provide customers with minimal required permissions:

**AWS IAM Policy** (for ECS + S3):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:CreateCluster",
        "ecs:CreateService",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:CreateTargetGroup",
        "ec2:CreateVpc",
        "ec2:CreateSubnet",
        "s3:CreateBucket",
        "s3:PutObject",
        "s3:GetObject",
        "ecr:GetAuthorizationToken",
        "ecr:CreateRepository",
        "ecr:PutImage"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateAccessKey",
        "iam:DeleteUser",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**GCP Service Account** (for Cloud Run + Storage):
```yaml
roles:
  - roles/run.admin
  - roles/storage.admin
  - roles/artifactregistry.admin
  - roles/iam.serviceAccountUser

conditions:
  - title: "Region restriction"
    expression: "resource.location == 'us-central1'"
```

## Code Security

### Source Code Protection

**GitHub Repository Access**:
- OAuth with minimal scopes (read-only for private repos)
- Temporary clone (deleted after build)
- Never stored permanently (only in object storage, encrypted)

**Code Scanning**:
```go
func ScanCode(repoPath string) ([]SecurityIssue, error) {
    issues := []SecurityIssue{}

    // Check for hardcoded secrets
    secrets := detectSecrets(repoPath)
    issues = append(issues, secrets...)

    // Check for dangerous dependencies
    vulns := scanDependencies(repoPath)
    issues = append(issues, vulns...)

    // Check for code injection risks
    injections := detectInjections(repoPath)
    issues = append(issues, injections...)

    return issues, nil
}
```

### Transformation Security

**Prevent Code Injection**:
```python
# Bad: String concatenation (vulnerable to injection)
def transform_call(call_info):
    code = f"s3.upload_file({call_info['source']}, {call_info['bucket']})"
    return code

# Good: AST manipulation (safe)
def transform_call(call_info):
    # Build AST nodes
    call_node = ast.Call(
        func=ast.Attribute(
            value=ast.Name(id='s3', ctx=ast.Load()),
            attr='upload_file',
            ctx=ast.Load()
        ),
        args=[
            ast.Constant(value=call_info['source']),
            ast.Constant(value=call_info['bucket'])
        ],
        keywords=[]
    )
    return ast.unparse(call_node)
```

**Validate Generated Code**:
```python
def validate_generated_code(code):
    try:
        # Parse to ensure valid Python
        tree = ast.parse(code)

        # Check for dangerous operations
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                # Block dangerous imports
                if any(name in BLOCKED_MODULES for name in node.names):
                    raise SecurityError(f"Blocked import: {node.names}")

            if isinstance(node, ast.Call):
                # Block dangerous functions
                if is_dangerous_call(node):
                    raise SecurityError(f"Blocked function call")

        return True
    except SyntaxError:
        raise ValidationError("Generated code is not valid Python")

BLOCKED_MODULES = ['os', 'subprocess', 'eval', '__import__']
```

## Network Security

### TLS Everywhere

**API**:
- TLS 1.3 only
- Strong cipher suites (ECDHE-RSA-AES256-GCM-SHA384)
- Certificate from Let's Encrypt (auto-renewed)

**Database Connections**:
```
postgresql://user:pass@db.infrar.io:5432/infrar?sslmode=require
```

**Redis (if used)**:
```
redis://default:password@redis.infrar.io:6379?tls=true
```

### Firewall Rules

**Ingress** (what can reach Infrar):
- Port 443 (HTTPS): From anywhere
- Port 22 (SSH): From office IP only (for admin access)
- Everything else: Blocked

**Egress** (what Infrar can reach):
- Port 443 (HTTPS): To GitHub, AWS, GCP, Azure APIs
- Port 5432 (PostgreSQL): To managed database only
- Everything else: Blocked (prevent data exfiltration)

### API Rate Limiting

Prevent abuse and DDoS:

```go
func RateLimitMiddleware() gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(100), 200) // 100 req/sec, burst 200

    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.JSON(429, gin.H{"error": "Rate limit exceeded"})
            c.Abort()
            return
        }
        c.Next()
    }
}
```

**Limits**:
- Anonymous: 10 req/min
- Authenticated: 100 req/min
- Premium: 1000 req/min

## Data Protection

### Encryption at Rest

**Database**:
- PostgreSQL encryption enabled
- Encrypted backups

**Object Storage**:
- Server-side encryption (AES-256)
- Customer-managed keys (optional)

**OpenTofu State Files**:
```hcl
terraform {
  backend "s3" {
    bucket         = "infrar-state"
    key            = "projects/${project_id}/terraform.tfstate"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID"
  }
}
```

### Data Retention

**Logs**: 90 days (30 days for MVP)
**Audit Trails**: 1 year
**Backups**: 30 days
**Deleted Projects**: Soft delete for 30 days, then hard delete

### GDPR Compliance

**User Rights**:
1. **Right to Access**: Export all user data via API
2. **Right to Deletion**: Delete account and all associated data
3. **Right to Portability**: Export data in JSON format
4. **Right to Rectification**: Update incorrect data

**Implementation**:
```go
// /api/users/me/export
func ExportUserData(c *gin.Context) {
    userID := c.GetString("user_id")

    // Gather all user data
    user := getUserData(userID)
    projects := getProjects(userID)
    deployments := getDeployments(userID)
    auditLogs := getAuditLogs(userID)

    data := map[string]interface{}{
        "user":        user,
        "projects":    projects,
        "deployments": deployments,
        "audit_logs":  auditLogs,
    }

    c.JSON(200, data)
}

// /api/users/me (DELETE)
func DeleteUser(c *gin.Context) {
    userID := c.GetString("user_id")

    // Soft delete (mark as deleted, retain for 30 days)
    softDeleteUser(userID)

    // Schedule hard delete after 30 days
    scheduleHardDelete(userID, time.Now().Add(30 * 24 * time.Hour))

    c.JSON(200, gin.H{"message": "Account scheduled for deletion"})
}
```

## Audit Logging

### What to Log

**Critical Events**:
- Authentication (login, logout, failed attempts)
- Authorization (permission denied)
- Credential access (who accessed which credentials)
- Deployments (who deployed what, where)
- Migrations (who migrated what)
- Configuration changes

**Log Format**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "credential_access",
  "user_id": "user-uuid",
  "user_email": "user@example.com",
  "resource_type": "provider_credentials",
  "resource_id": "cred-uuid",
  "action": "retrieve",
  "result": "success",
  "ip_address": "1.2.3.4",
  "user_agent": "Mozilla/5.0...",
  "metadata": {
    "provider": "aws",
    "purpose": "deployment",
    "deployment_id": "deploy-uuid"
  }
}
```

### Log Analysis

**Alerting Rules**:
```
Alert: Multiple failed login attempts
Condition: 5 failed logins from same IP in 5 minutes
Action: Block IP, notify security team

Alert: Unusual credential access
Condition: Credentials accessed outside business hours
Action: Notify user, require re-authentication

Alert: Mass data export
Condition: User exports > 100 projects in 1 hour
Action: Rate limit, notify security team
```

## Incident Response

### Security Incident Playbook

**Phase 1: Detection**
- Automated alerts (from logs, monitoring)
- User reports
- Security scans

**Phase 2: Containment**
- Isolate affected systems
- Rotate compromised credentials
- Block malicious IPs
- Disable compromised accounts

**Phase 3: Investigation**
- Analyze logs
- Determine scope (what was accessed?)
- Identify attack vector

**Phase 4: Remediation**
- Patch vulnerabilities
- Restore from backups if needed
- Reset credentials

**Phase 5: Communication**
- Notify affected users (within 72 hours per GDPR)
- Provide transparency report
- Offer remediation steps

**Phase 6: Post-Mortem**
- Document lessons learned
- Implement preventive measures
- Update runbooks

### Breach Notification

If customer credentials compromised:

1. **Immediate** (< 1 hour):
   - Rotate credentials in Infrar
   - Block access to affected resources

2. **Within 24 hours**:
   - Email affected customers
   - Provide incident details
   - Offer support

3. **Within 72 hours**:
   - Full transparency report
   - Regulatory notification (GDPR)

## Security Testing

### Automated Testing

**Unit Tests**:
```go
func TestAuthenticationRequired(t *testing.T) {
    router := setupRouter()

    // Request without token
    req, _ := http.NewRequest("GET", "/api/projects", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    // Should return 401
    assert.Equal(t, 401, w.Code)
}

func TestAuthorizationEnforced(t *testing.T) {
    router := setupRouter()

    // User A tries to access User B's project
    token := generateJWT(userA)
    req, _ := http.NewRequest("GET", "/api/projects/" + userBProject, nil)
    req.AddCookie(&http.Cookie{Name: "auth_token", Value: token})
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    // Should return 403
    assert.Equal(t, 403, w.Code)
}
```

**Integration Tests**:
```python
def test_credential_encryption():
    # Store credentials
    store_credentials(user_id, provider, credentials)

    # Retrieve from database directly
    raw = db.query("SELECT encrypted_credentials FROM provider_credentials WHERE user_id = ?", user_id)

    # Verify it's encrypted (not plaintext)
    assert credentials not in raw
    assert is_encrypted(raw)

    # Verify decryption works
    retrieved = retrieve_credentials(user_id, provider)
    assert retrieved == credentials
```

### Manual Testing

**Penetration Testing**:
- SQL injection testing
- XSS testing
- CSRF testing
- Authentication bypass attempts
- Authorization escalation attempts

Performed quarterly by external security firm.

**Security Audit**:
- Code review by security team
- Dependency vulnerability scanning
- Infrastructure hardening review

Performed before each major release.

## Compliance

### SOC 2 Type II (Target)

**Security Principle Compliance**:
- ✅ Logical access controls
- ✅ Encryption
- ✅ Change management
- ✅ Incident response
- ✅ Monitoring

### ISO 27001 (Target)

**Key Controls**:
- Information security policies
- Asset management
- Access control
- Cryptography
- Operations security
- Communications security

## Security Roadmap

### MVP (Phase 1)
- ✅ TLS everywhere
- ✅ JWT authentication
- ✅ RBAC authorization
- ✅ Credential encryption (PostgreSQL)
- ✅ Basic audit logging

### Production (Phase 2)
- ⬜ HashiCorp Vault integration
- ⬜ Advanced rate limiting
- ⬜ IP allowlisting
- ⬜ 2FA/MFA support
- ⬜ Security scanning (SAST/DAST)

### Enterprise (Phase 3)
- ⬜ SSO (SAML, OIDC)
- ⬜ SOC 2 Type II certification
- ⬜ ISO 27001 certification
- ⬜ Customer-managed encryption keys
- ⬜ Private cloud deployment

## Conclusion

Security at Infrar is:
- **Multi-layered**: Defense in depth
- **Proactive**: Security by design
- **Transparent**: Audit trails and reporting
- **Compliant**: GDPR, SOC 2 (target)
- **Evolving**: Continuous improvement

Security is everyone's responsibility. Report security issues to: security@infrar.io (PGP key available).
