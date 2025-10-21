# Infrar Open Core Model

**Last Updated**: October 2025

## Overview

Infrar uses an **open core model** where the transformation engine and core functionality are fully open source (Apache 2.0), while advanced intelligence features and enterprise capabilities remain proprietary.

**Rationale**: Infrastructure tools require deep trust. Open sourcing the core transformation engine removes adoption barriers while enabling us to build sustainable business around high-value features.

---

## What's Open Source (Apache 2.0)

### Core Transformation Engine
**Repository**: [infrar-engine](https://github.com/QodeSrl/infrar-engine)

The complete AST-based code transformation system:
- Python parser and AST analysis
- Transformation rule engine
- Code generation for provider SDKs
- Plugin system architecture
- Validation and testing framework

**Why Open**: Security teams must audit code that transforms their infrastructure code. Transparency builds trust and enables community contributions.

### Cloud Provider Plugins
**Repository**: [infrar-plugins](https://github.com/QodeSrl/infrar-plugins)

Transformation rules for all major cloud providers:
- Storage operations (S3, Cloud Storage, Blob Storage)
- Compute primitives (Lambda, Cloud Functions, Azure Functions)
- Database operations (RDS, Cloud SQL, Azure SQL)
- Messaging (SQS, Pub/Sub, Service Bus)
- Networking (VPC, security groups, load balancers)

**Why Open**: Community can maintain provider API changes and contribute new cloud providers. Ecosystem benefits from standardized abstractions.

### CLI Tool
**Repository**: [infrar-cli](https://github.com/QodeSrl/infrar-cli)

Command-line interface for local development:
- Project initialization
- Code transformation
- Provider authentication
- Deployment commands
- Configuration management

**Why Open**: Developers need local tooling for development without barriers. Standard practice for infrastructure tools (Terraform, Kubernetes, etc.).

### Language SDKs
**Repositories**: [infrar-sdk-python](https://github.com/QodeSrl/infrar-sdk-python), infrar-sdk-nodejs, etc.

The provider-agnostic SDKs developers write code against:
```python
from infrar.storage import upload, download
from infrar.compute import deploy_function
from infrar.database import query
```

**Why Open**: Developers write application code against these SDKs. They must be freely available for adoption.

### Documentation
**Repository**: [infrar-docs](https://github.com/QodeSrl/infrar-docs)
**License**: Creative Commons BY 4.0

Complete documentation including:
- Getting started guides
- API reference
- Architecture documentation
- Plugin development guides

---

## What's Proprietary

### Cost Intelligence Engine
Advanced cost comparison and optimization:
- Real-time pricing data across AWS, GCP, Azure
- Cost comparison algorithms
- Optimization recommendations
- Forecasting and trend analysis

**Rationale**: Significant investment in data collection and maintenance. Core differentiator that enterprises will pay for.

### Brownfield Discovery
Scan and analyze existing cloud infrastructure:
- Auto-discover resources in existing accounts
- Map resources to Infrar capabilities
- Analyze code coupling to cloud SDKs
- Generate migration difficulty scores

**Rationale**: Complex enterprise feature requiring ongoing maintenance across provider APIs.

### Migration Planning Tools
Sophisticated planning and ROI analysis:
- Multi-scenario migration planning
- Cost-benefit analysis with timelines
- Risk assessment and mitigation
- Partial migration planning
- ROI calculation

**Rationale**: High-value enterprise feature with sophisticated algorithms.

### Web Platform
Managed SaaS platform with various tiers:
- Basic free tier for evaluation
- Professional tier for teams
- Enterprise tier with advanced features

**Rationale**: Hosting costs and managed service value justify SaaS model.

### Enterprise Features
Standard enterprise requirements:
- RBAC and SSO/SAML
- Audit logs and compliance reports
- Multi-account management
- Team collaboration features
- SLA guarantees

**Rationale**: Enterprise buyers expect these features and will pay premium pricing.

---

## Why Open Core?

### Benefits of Open Source Core

**Trust**: Security teams can audit transformation logic
**Adoption**: No barriers to trying and using the core technology
**Community**: Contributors improve quality and add providers
**Validation**: Academic research and industry validation
**Portability**: No lock-in to Infrar itself (ironic but important!)

### Benefits of Proprietary Intelligence

**Sustainability**: Revenue supports ongoing development and maintenance
**Investment**: Complex features require dedicated engineering
**Data Quality**: Professional team maintains pricing data and algorithms
**Support**: Paying customers get SLAs and dedicated help

### Comparable Models

This model is proven by successful infrastructure companies:
- **GitLab**: Open core for CI/CD platform
- **Elastic**: Open search engine, proprietary enterprise features
- **MongoDB**: Open database, proprietary enterprise and cloud
- **HashiCorp**: Open Terraform/Vault, proprietary enterprise features
- **Red Hat**: Open RHEL, subscription for support and enterprise

---

## Licensing

### Open Source Components
**License**: Apache License 2.0

All transformation engine, CLI, plugins, and SDKs use Apache 2.0:
- Permissive for commercial use
- Explicit patent grant
- Industry standard for infrastructure tools
- Compatible with proprietary components

### Proprietary Components
**License**: Commercial subscription

Web platform, intelligence features, and enterprise capabilities require paid subscription.

### Documentation
**License**: Creative Commons Attribution 4.0 (CC BY 4.0)

Documentation is freely available with attribution. Code examples in documentation use Apache 2.0.

---

## Contributing

We welcome contributions to all open source components:
- Transformation rules for cloud providers
- New plugin development
- Bug fixes and improvements
- Documentation enhancements
- Language SDK implementations

See individual repository contributing guidelines for details.

---

## Frequently Asked Questions

### Can I use Infrar for free?

Yes! The entire transformation engine, CLI, plugins, and SDKs are open source (Apache 2.0). You can:
- Transform code to any cloud provider
- Self-host all deployments
- Use in commercial projects
- Modify and redistribute

### What do I need to pay for?

Paid features are optional and focus on convenience and intelligence:
- Hosted web platform (managed service)
- Cost comparison across providers
- Brownfield discovery for existing infrastructure
- Migration planning and ROI tools
- Enterprise features (SSO, RBAC, audit logs)

### Can I self-host everything?

Yes! All core transformation functionality is open source. The paid features are primarily:
- Hosted/managed versions of open source tools
- Intelligence layer (cost data, analysis)
- Enterprise-specific features

### Will core features stay open source?

Yes. We're committed to keeping the transformation engine, plugins, CLI, and SDKs fully open source under Apache 2.0. This is fundamental to building trust in infrastructure tooling.

### What if I need a feature that's proprietary?

You have options:
1. Build it yourself using the open APIs
2. Subscribe to access proprietary features
3. Request it as an open source contribution
4. Engage our professional services for custom development

---

**For detailed architecture and technical documentation, see [infrar-docs](https://github.com/QodeSrl/infrar-docs).**
