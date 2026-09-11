---
description: "![Shipper Banner](https://raw.githubusercontent.com/shippercli/assets/main/banner.png)"
---

# Shipper CLI Specifications

![Shipper Banner](https://raw.githubusercontent.com/shippercli/assets/main/banner.png)

Detailed specifications for Shipper CLI features and architecture.

## Specifications

### Core Features

- [CLI Commands](./cli-commands.md) - Command definitions and usage
- [Configuration System](./configuration-system.md) - shipper.yml schema
- [Deployment Execution](./deployment-execution.md) - Apply/plan workflow
- [Plan/Apply Workflow](./plan-apply-workflow.md) - Declarative deployment
- [Provider Architecture](./provider-architecture.md) - Plugin system
- [Site Lifecycle](./site-lifecycle.md) - Creation, update, deletion

### Provider-Specific

- [Ploi Provider](./ploi-provider.md) - Ploi.io integration
- [Forge Provider](./forge-provider.md) - Laravel Forge integration

### Advanced Features

- [Build System](./build-system.md) - PHAR binary creation
- [GitHub Actions](./github-actions.md) - CI/CD workflows
- [Database Management](./MAR-22-database-management.md) - DB provisioning
- [Environment Variables](./MAR-23-environment-variables.md) - Variable management
- [Deploy Scripts](./MAR-32-deploy-scripts.md) - Deployment automation
- [Domain Aliases](./domain-aliases.md) - Multi-domain support
- [SSL Management](./ssl-management.md) - Certificate handling

### Architecture

- [Strict Type System](./strict-type-system.md) - Type safety standards
- [Variable Interpolation](./variable-interpolation.md) - Config templating
- [Security Hardening](./security-hardening.md) - Security practices

## License

MIT