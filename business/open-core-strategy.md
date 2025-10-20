# Infrar Open Core Strategy

**Document Status**: Final
**Last Updated**: October 2025
**Decision**: Open Core Model

## Executive Summary

Infrar will adopt an **open core model** where the transformation engine and core functionality are fully open source (Apache 2.0), while intelligence features and enterprise capabilities remain proprietary.

**Rationale**: Infrastructure tools require deep trust. Open sourcing the core transformation engine removes adoption barriers while protecting high-value differentiators (cost intelligence, brownfield discovery, enterprise features).

## What's Open Source

### Core Transformation Engine
**License**: Apache 2.0
**Repository**: `infrar/infrar-engine`

Components:
- AST parsing and code analysis
- Transformation rule engine
- Code generation for provider SDKs
- Plugin system architecture
- Capability abstraction framework

**Why Open**:
- Security teams must audit code transformation logic
- Community can verify correctness and contribute improvements
- Removes fear of vendor lock-in (ironic for anti-lock-in tool)
- Enables academic research and validation

### Core Plugins
**License**: Apache 2.0
**Repositories**: `infrar/plugins-*`

**Included Plugins**:
- `infrar-storage`: Object storage operations (S3, GCS, Azure Blob)
- `infrar-compute`: Compute primitives (Lambda, Cloud Functions, Azure Functions)
- `infrar-database`: Relational database operations (RDS, Cloud SQL, Azure SQL)
- `infrar-messaging`: Queue and pub/sub operations (SQS, Pub/Sub, Service Bus)
- `infrar-network`: VPC, security groups, load balancers

**Why Open**:
- Community can maintain provider API changes
- Contributors can add new cloud providers
- Ecosystem benefits from standardized abstractions
- Network effects improve quality faster

### CLI Tool
**License**: Apache 2.0
**Repository**: `infrar/infrar-cli`

Features:
- Project initialization
- Local transformation and testing
- Provider authentication
- Basic deployment commands
- Configuration management

**Why Open**:
- Developers need local tooling for development
- Reduces friction to adoption
- Community can integrate with other tools
- Standard for infrastructure CLIs (terraform, kubectl, etc.)

### Documentation
**License**: Creative Commons BY 4.0
**Repository**: `infrar/infrar-docs`

Includes:
- Getting started guides
- API reference documentation
- Transformation rule specifications
- Plugin development guides
- Architecture documentation

### SDKs & Libraries
**License**: Apache 2.0
**Repositories**: `infrar/sdk-python`, `infrar/sdk-nodejs`, etc.

The provider-agnostic SDKs that developers import:
```python
from infrar.storage import upload, download
from infrar.compute import deploy_function
from infrar.database import query
```

**Why Open**:
- Developers write code against these SDKs
- Must be freely available for adoption
- Community can contribute language support

## What's Proprietary/Paid

### 1. Cost Intelligence Engine
**Pricing**: Included in paid tiers only
**Technology**: Proprietary

Components:
- Real-time pricing database (AWS, GCP, Azure, updated daily)
- Cost comparison algorithms
- Optimization recommendation engine
- Forecasting and trend analysis
- Cost anomaly detection
- Multi-cloud cost allocation

**Why Proprietary**:
- Significant investment in data collection and maintenance
- Core competitive differentiator
- Enterprise buyers will pay for this
- Data network effects (more users = better insights)

**Value Proposition**: Save 15-40% on cloud costs through intelligent provider selection.

### 2. Brownfield Discovery
**Pricing**: Enterprise tier only
**Technology**: Proprietary

Features:
- Scan existing cloud accounts (read-only)
- Auto-discover resources and relationships
- Map resources to infrar capabilities
- Analyze code coupling to cloud SDKs
- Generate migration difficulty scores
- Identify quick wins and cost savings opportunities

**Why Proprietary**:
- Complex integration with provider APIs
- High-value enterprise feature
- Requires ongoing maintenance per provider
- Justifies enterprise pricing

**Value Proposition**: Understand what you're running and what migration would take.

### 3. Migration Planning & ROI Calculator
**Pricing**: Professional and Enterprise tiers
**Technology**: Proprietary

Features:
- Multi-scenario migration planning
- Cost-benefit analysis with timelines
- Risk assessment and mitigation strategies
- Partial migration planning (capability by capability)
- Rollback planning
- Dependency mapping
- ROI calculation with break-even analysis

**Why Proprietary**:
- Sophisticated algorithms and business logic
- High perceived value for enterprise buyers
- Differentiates from DIY open source usage

**Value Proposition**: Plan migrations with confidence, know the ROI upfront.

### 4. Web Platform UI
**Pricing**: Tiered (Basic Free, Pro Paid, Enterprise Paid)
**Technology**: Proprietary (with potential open source basic version)

**Free Tier Features**:
- Single project management
- Manual transformations
- Basic deployment history
- Community support

**Pro Tier Features** ($99-499/month):
- Unlimited projects
- Automated GitHub integration
- Real-time cost comparison
- Multi-cloud deployment dashboard
- Deployment rollback
- Email support

**Enterprise Tier Features** ($2K-10K+/month):
- Everything in Pro
- Brownfield discovery
- Migration planning tools
- Multi-account management
- Team collaboration (RBAC)
- Audit logs and compliance reports
- SSO/SAML
- SLA guarantees
- Dedicated support

**Why Proprietary**:
- Significant development and hosting costs
- Natural SaaS monetization point
- Customers expect managed services
- Reduces barrier to self-hosting for advanced users

### 5. Multi-Account Management
**Pricing**: Enterprise tier only
**Technology**: Proprietary

Features:
- Manage multiple AWS/GCP/Azure accounts
- Centralized credential management
- Organization-wide cost visibility
- Policy enforcement across accounts
- Consolidated billing analysis

**Why Proprietary**:
- Enterprise-specific feature
- Complex security requirements
- Justifies higher pricing tier

### 6. Team & Collaboration Features
**Pricing**: Pro and Enterprise tiers
**Technology**: Proprietary

Features:
- User management and RBAC
- Deployment approvals and workflows
- Change notifications
- Activity audit logs
- Team cost allocation
- Shared capability libraries

**Why Proprietary**:
- Standard SaaS features
- Enterprise buyers expect these
- Requires platform infrastructure

### 7. Enterprise Support & SLAs
**Pricing**: Enterprise tier only
**Service**: Proprietary

Includes:
- 99.9% uptime SLA
- Priority bug fixes
- Dedicated support engineer
- Migration consulting hours
- Custom transformation rule development
- Training sessions

## Monetization Model

### Pricing Tiers

#### Free (Open Source + Self-Hosted)
**Price**: $0
**Target**: Individual developers, startups, evaluation

**Includes**:
- Full transformation engine (self-compiled)
- All core plugins
- CLI tool
- Self-hosted deployments
- Community support (GitHub Discussions)
- Documentation

**Limitations**:
- No web UI (CLI only)
- No cost intelligence
- No brownfield discovery
- Self-managed infrastructure
- No SLA

---

#### Starter (Hosted)
**Price**: $49/month
**Target**: Small teams, side projects

**Includes**:
- Everything in Free
- Web UI access (basic)
- 1 team member
- 5 projects
- 100 deployments/month
- Email support (48hr response)
- Hosted transformation service

**Limitations**:
- No cost intelligence
- No brownfield discovery
- No team features

---

#### Professional
**Price**: $299/month
**Target**: Growing startups, small teams

**Includes**:
- Everything in Starter
- Up to 5 team members
- Unlimited projects
- Unlimited deployments
- **Cost intelligence** (comparison across providers)
- **Basic migration planning** (single capability)
- GitHub/GitLab integration
- Automated deployments
- Deployment rollback
- Priority email support (24hr response)
- Slack/Discord community access

---

#### Enterprise
**Price**: Starting at $2,000/month (custom pricing)
**Target**: Large companies, multiple teams, complex migrations

**Includes**:
- Everything in Professional
- Unlimited team members
- **Full brownfield discovery** (scan existing accounts)
- **Advanced migration planning** (multi-capability, ROI analysis)
- **Multi-account management**
- **Custom transformation rules**
- RBAC and SSO/SAML
- Audit logs and compliance reports
- 99.9% uptime SLA
- Dedicated support engineer
- Priority bug fixes
- Migration consulting (10 hours included)
- Training sessions
- Private plugin registry
- On-premise deployment option

**Add-ons**:
- Additional consulting hours: $200/hour
- Custom plugin development: $5K-20K per plugin
- White-label deployment: Custom pricing

---

### Additional Revenue Streams

#### 1. Cloud Marketplace Listings
- AWS Marketplace: Pay-as-you-go billing through AWS account
- Azure Marketplace: Integration with Azure committed spend
- GCP Marketplace: GCP billing integration

**Benefits**:
- Enterprises have pre-committed cloud spend
- Reduces procurement friction
- 3-5% marketplace fee, but higher conversion

#### 2. Professional Services
**Pricing**: $150-300/hour

Services:
- Migration consulting and planning
- Custom capability development
- Transformation rule optimization
- Multi-cloud architecture design
- Training and workshops
- Code audit and coupling analysis

**Target**: Enterprise customers with complex migrations

#### 3. Partner Program
**Commission**: 20-30% recurring

Partners:
- Cloud consultancies (Accenture, Deloitte, etc.)
- System integrators
- Managed service providers

**Value**: Expand reach to enterprise market through trusted advisors

#### 4. Training & Certification
**Pricing**: $500-2,000 per course

Programs:
- Infrar Developer Certification
- Multi-Cloud Architecture with Infrar
- Enterprise Migration Planning
- Plugin Development Workshop

## Go-to-Market Implications

### Open Source Community
**Investment**: 1-2 FTE dedicated to community

Activities:
- Maintain open source repositories
- Review and merge community PRs
- Host monthly community calls
- Sponsor conferences and meetups
- Create tutorial content
- Answer GitHub Discussions

**Goal**: Build trust and adoption, create evangelists

### Developer Marketing
**Channels**:
- GitHub (star campaigns, trending)
- Hacker News, Reddit, dev.to
- Conference talks (KubeCon, AWS re:Invent, Google Next)
- Blog (technical deep-dives)
- YouTube (tutorials, demos)

**Message**: "Try it free, see the transformation yourself"

### Enterprise Sales
**Channels**:
- Direct sales team (once PMF established)
- Cloud marketplaces
- Partner ecosystem
- Trade shows (Gartner, AWS Summits)

**Message**: "Reduce cloud costs 20-30%, eliminate vendor lock-in"

### Product-Led Growth
**Funnel**:
1. Download CLI / Try web UI → **Free**
2. Deploy first project → **Activation**
3. Compare costs across providers → **Aha moment**
4. Add team members → **Upgrade to Pro**
5. Discover existing infrastructure → **Upgrade to Enterprise**
6. Plan first migration → **Retention**

## Licensing Strategy

### Open Source License: Apache 2.0

**Why Apache 2.0 over MIT**:
- Explicit patent grant (protects community and commercial users)
- Permissive enough for enterprise adoption
- Compatible with proprietary components
- Industry standard for infrastructure tools

**Components under Apache 2.0**:
- `infrar-engine` (transformation engine)
- `infrar-cli` (command-line tool)
- `infrar-plugins-*` (all core plugins)
- `infrar-sdk-*` (language SDKs)

### Proprietary License

**Components**:
- Web platform UI (partially - basic may be open)
- Cost intelligence database and engine
- Brownfield discovery service
- Migration planning algorithms
- Enterprise features (RBAC, SSO, audit logs)

**License Model**: Subscription-based, per-user or per-account

### Contributor License Agreement (CLA)

**Requirement**: Contributors to open source repos must sign CLA

**Purpose**:
- Allows us to relicense if needed (unlikely, but option)
- Protects against patent trolls
- Standard practice (Google, Microsoft, etc.)

**Type**: Individual CLA (simple, automated via CLA Assistant)

## Competitive Protection

### Even with Open Source Core, We Maintain Moat Through:

1. **Data Network Effects**: More users → better cost data → better recommendations
2. **Brand & Trust**: First mover, established community
3. **Hosting & Convenience**: Most won't self-host complex systems
4. **Intelligence Layer**: Proprietary algorithms and data
5. **Enterprise Features**: Difficult to replicate (SSO, RBAC, compliance)
6. **Professional Services**: Deep expertise and relationships
7. **Ecosystem**: Plugins, integrations, partnerships

### What If Cloud Providers Copy Us?

**Likely Response**: They will try, but face conflicts of interest

**Our Advantages**:
- **Neutrality**: We recommend best provider, they can't
- **Multi-cloud**: We manage cross-cloud complexity
- **Community trust**: Open source builds credibility
- **Speed**: We move faster than large orgs

**Precedent**:
- AWS launched many open-source-inspired services
- Original projects often maintain leadership (MongoDB, Elastic, Redis)
- Enterprise buyers prefer neutral third parties

## Success Metrics

### Open Source Health (12-month targets)
- 10K+ GitHub stars
- 500+ monthly active contributors
- 50+ external code contributors
- 20+ community plugins
- 100K+ CLI downloads

### Commercial Health (Year 1 targets)
- 100 paying customers
- $500K ARR
- 10 enterprise customers ($2K+ monthly)
- 90% gross margin
- <5% monthly churn

### Technical Metrics
- 99.9% transformation success rate
- <1s average transformation time
- Support for 3 languages (Python, Node.js, Go)
- 15+ capabilities defined
- 3 cloud providers (AWS, GCP, Azure)

## Risk Mitigation

### Risk: Competitors fork and compete
**Mitigation**:
- Intelligence layer is proprietary
- Brand and community matter more than code
- Network effects from cost data
- Speed of iteration

### Risk: Large cloud providers build similar tools
**Mitigation**:
- We're neutral, they have conflict of interest
- Open source community trusts us more
- Multi-cloud orchestration is complex
- Enterprise buyers prefer independent vendors

### Risk: Open source cannibalizes paid revenue
**Mitigation**:
- Self-hosting is complex (most will pay)
- Value is in intelligence, not just transformation
- Enterprise features justify pricing
- Hosting/convenience is worth premium

### Risk: Community demands features stay open
**Mitigation**:
- Clear separation from day 1
- Apache 2.0 allows commercial use
- Precedent: GitLab, Elastic, MongoDB succeeded
- Intelligence layer naturally proprietary

## Decision Log

**Date**: October 2025
**Decision**: Adopt open core model with Apache 2.0 for engine/plugins, proprietary for intelligence
**Rationale**: Maximizes adoption while protecting high-value differentiation
**Alternatives Considered**:
1. Fully proprietary: Rejected due to trust barriers in infrastructure
2. Fully open source: Rejected due to unclear monetization path
3. BSL (Business Source License): Rejected due to community backlash risk

**Approval**: Founder

---

## Next Steps

1. **Create open source repositories** (engine, CLI, plugins)
2. **Draft contributor guidelines** and CLA
3. **Set up community channels** (Discord, GitHub Discussions)
4. **Build MVP** following open core split
5. **Launch** with open source first, paid features in Phase 3
6. **Measure** community growth and conversion metrics

---

**Document Owner**: Product & Strategy
**Review Cycle**: Quarterly (adjust based on learnings)
