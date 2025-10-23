# Infrar Documentation

Official documentation for the **Infrar Infrastructure Intelligence Platform**.

## What is Infrar?

Infrar is a multi-cloud infrastructure platform that transforms cloud-agnostic code into provider-specific implementations. It enables developers to write code once and deploy to AWS, GCP, Azure, or other cloud providers without vendor lock-in.

## 🚀 Quick Start

- **New to Infrar?** Start with [Getting Started](getting-started/dev-environment-setup.md)
- **Want to contribute?** See [CONTRIBUTING.md](CONTRIBUTING.md)
- **Understanding the architecture?** Read [Architecture Overview](architecture/overview.md)

## 📚 Documentation

### Core Concepts

- **[Capabilities](concepts/capabilities.md)** - Infrastructure capabilities abstraction
- **[Provider Plugins](concepts/plugins.md)** - How the plugin system works

### Architecture

- **[Overview](architecture/overview.md)** - High-level system design
- **[Repository Structure](architecture/repository-structure.md)** - Codebase organization
- **[Authentication Plugins](architecture/authentication-plugins.md)** - Multi-method authentication system
- **[Template Engine](architecture/template-engine.md)** - Template-driven infrastructure generation
- **[Multi-Cloud Orchestration](architecture/multi-cloud-orchestration.md)** - Cross-cloud deployment

### Development

- **[Getting Started](getting-started/dev-environment-setup.md)** - Set up your development environment
- **[Adding a Provider](CONTRIBUTING.md#adding-a-new-cloud-provider)** - How to add a new cloud provider
- **[Testing Guide](development/testing/)** - Testing strategies and best practices

### User Guides

- **[Greenfield Deployments](user-guides/greenfield.md)** - Deploy new projects from scratch

### API Reference

- **[Storage API](api/infrar-storage.md)** - Object storage operations
- **[Frontend Guide](api/frontend-guide.md)** - Frontend integration

## 🏗️ Repository Structure

Infrar is organized into multiple repositories following an open-core model:

| Repository | Description | License | Status |
|------------|-------------|---------|--------|
| **infrar-engine** | Core transformation engine | GPL v3 | ✅ Active |
| **infrar-sdk-python** | Python SDK for developers | GPL v3 | ✅ Active |
| **infrar-plugins** | Provider plugins (AWS, GCP, Azure) | GPL v3 | ✅ Active |
| **infrar-cli** | Command-line interface | GPL v3 | ✅ Active |
| **infrar-docs** | Documentation (this repo) | GPL v3 | ✅ Active |
| **infrar-platform** | Backend API services | Proprietary | 🚧 Development |
| **infrar-web** | Web frontend | Proprietary | 🚧 Development |

See [Repository Structure](architecture/repository-structure.md) for detailed information.

## 🎯 Current Status

**Phase**: MVP Development

**Completed**:
- ✅ Provider plugin system architecture
- ✅ Authentication plugin system
- ✅ Template-based infrastructure generation
- ✅ GCP provider with storage support
- ✅ AWS provider foundation
- ✅ Azure provider foundation

**In Progress**:
- 🚧 Platform backend services
- 🚧 Web frontend interface
- 🚧 Additional service capabilities

## 🤝 Contributing

We welcome contributions! The easiest way to contribute is by adding support for new cloud providers or expanding existing provider capabilities.

**Quick links**:
- [How to add a cloud provider](CONTRIBUTING.md#adding-a-new-cloud-provider)
- [How to add a service](CONTRIBUTING.md#adding-a-new-service-to-an-existing-provider)
- [Development guidelines](CONTRIBUTING.md#plugin-development-guidelines)

## 📖 Key Documents

| Document | Purpose |
|----------|---------|
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute to Infrar |
| [Architecture Overview](architecture/overview.md) | System architecture and design |
| [Plugin System](concepts/plugins.md) | How provider plugins work |
| [Repository Structure](architecture/repository-structure.md) | Code organization |

## 🔗 Links

- **GitHub Organization**: [github.com/QodeSrl](https://github.com/QodeSrl)
- **Main Repository**: [infrar-engine](https://github.com/QodeSrl/infrar-engine)
- **Plugin Repository**: [infrar-plugins](https://github.com/QodeSrl/infrar-plugins)

## 📝 License

- Documentation: Creative Commons Attribution 4.0 International (CC BY 4.0)
- Code examples: GPL v3 (matching code repositories)

---

**Questions?** Check the [concepts](concepts/) or [architecture](architecture/) sections, or open an issue in the relevant repository.
