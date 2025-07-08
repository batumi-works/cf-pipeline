# Cloudflare Workers Central CI/CD Pipeline

A comprehensive, centralized CI/CD platform for Cloudflare Workers that provides reusable GitHub workflows, shared npm packages, and security-first deployment practices.

## Architecture Overview

This repository contains:
- **Reusable GitHub Actions workflows** for CI/CD
- **Composite actions** for consistent environment setup
- **Security configurations** with OIDC authentication
- **Observability integrations** with Datadog

## Usage

### For New Projects

Use the template repository:
```bash
npx create-cloudflare-worker@latest my-worker --template batumi-works/cloudflare-worker-template
```

### For Existing Projects

1. Install the shared scripts:
```bash
npm install @batumi/cf-scripts
```

2. Add the workflow to your repository:
```yaml
# .github/workflows/ci.yml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: batumi-works/cf-pipeline/.github/workflows/ci.yml@v1
    with:
      node_version: '18'
      wrangler_version: 'latest'
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      DATADOG_API_KEY: ${{ secrets.DATADOG_API_KEY }}
```

## Features

- **Sub-10s local testing** with Miniflare and workerd
- **Sub-60s deployments** with optimized caching
- **Zero-config security** with OIDC authentication
- **Comprehensive observability** with Datadog integration
- **Rollback capability** for incident response

## Security

- OIDC-based authentication (no long-lived secrets)
- Automated security scanning (SAST, dependency checks)
- API token scoping and rotation
- Compliance reporting and audit trails

## Performance Targets

- Local CI runs: ≤10s
- Remote deployments: ≤60s
- Repository configuration: ≤1kB per Worker repo
- Rollback time: <5min

## Documentation

- [Getting Started](./docs/getting-started.md)
- [Security Best Practices](./docs/security.md)
- [Troubleshooting](./docs/troubleshooting.md)
- [Migration Guide](./docs/migration.md)

## License

MIT License - see [LICENSE](LICENSE) for details.