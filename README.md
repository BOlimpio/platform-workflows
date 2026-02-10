# Platform Workflows

Centralized, reusable GitHub Actions workflow templates for Terraform infrastructure management.

## 🏗️ Overview

This repository provides enterprise-grade, reusable workflow templates that project repositories can consume. This ensures:

- **Consistency** - All projects follow the same CI/CD patterns
- **Security** - Centralized security and compliance controls
- **Maintainability** - Update once, apply everywhere
- **Governance** - Platform team controls the standards

## 📦 Available Workflows

| Workflow | Description | Trigger |
|----------|-------------|---------|
| `terraform-ci.yml` | CI pipeline with format, validate, lint, test, security, compliance | `workflow_call` |
| `terraform-deploy.yml` | Deploy pipeline with plan, approval, apply | `workflow_call` |
| `terraform-destroy.yml` | Destroy pipeline with double approval gate | `workflow_call` |

## 🚀 Quick Start

### 1. In Your Project Repository

Create `.github/workflows/terraform.yml`:

```yaml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      test_mode:
        description: 'Test mode'
        type: choice
        default: 'environment'
        options: ['environment', 'modules']

jobs:
  # CI Pipeline
  ci:
    uses: YOUR-ORG/platform-workflows/.github/workflows/terraform-ci.yml@v1
    with:
      working_directory: "environments/dev"
      environment: "dev"
      test_mode: ${{ github.event.inputs.test_mode || 'environment' }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}

  # Deploy Pipeline (main branch only)
  deploy:
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: YOUR-ORG/platform-workflows/.github/workflows/terraform-deploy.yml@v1
    with:
      working_directory: "environments/dev"
      environment: "dev"
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

### 2. Required Secrets

Configure these in your project repository:

| Secret | Description | Required |
|--------|-------------|----------|
| `AWS_ROLE_ARN` | IAM Role ARN for OIDC | ✅ |
| `INFRACOST_API_KEY` | Infracost API key | Optional |

### 3. Required Environments

Create these GitHub environments with protection rules:

| Environment | Purpose | Reviewers |
|-------------|---------|-----------|
| `dev` | Development deployments & destroy | Optional |
| `staging` | Staging deployments & destroy | 1+ Required |
| `production` | Production deployments & destroy | 2+ Required |

> **Note**: The destroy workflow includes a built-in double approval gate for additional safety.

## 📋 Workflow Reference

### terraform-ci.yml

Full CI pipeline with all quality gates.

#### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `working_directory` | string | ✅ | - | Terraform working directory |
| `environment` | string | ✅ | - | Environment name |
| `terraform_version` | string | ❌ | `1.9.0` | Terraform version |
| `tflint_version` | string | ❌ | `v0.50.0` | TFLint version |
| `test_mode` | string | ❌ | `environment` | `environment` or `modules` |
| `modules_to_test` | string | ❌ | `all` | Comma-separated modules |
| `modules_base_path` | string | ❌ | `modules/aws` | Path to modules |
| `run_apply_tests` | boolean | ❌ | `false` | Run apply tests |
| `enable_security_scan` | boolean | ❌ | `true` | Enable Checkov |
| `enable_compliance` | boolean | ❌ | `true` | Enable terraform-compliance |
| `enable_cost_estimation` | boolean | ❌ | `true` | Enable Infracost |
| `enable_lint` | boolean | ❌ | `true` | Enable TFLint |
| `compliance_features_path` | string | ❌ | `compliance/features` | Compliance rules path |
| `checkov_config_file` | string | ❌ | `.checkov.yaml` | Checkov config |
| `tflint_config_file` | string | ❌ | `.tflint.hcl` | TFLint config |
| `aws_region` | string | ❌ | `us-east-1` | AWS region |

#### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AWS_ROLE_ARN` | ✅ | OIDC IAM Role ARN |
| `INFRACOST_API_KEY` | ❌ | Infracost API key |

#### Outputs

| Output | Description |
|--------|-------------|
| `format_result` | Format check result |
| `validate_result` | Validation result |
| `lint_result` | Lint result |
| `test_result` | Test result |
| `security_result` | Security scan result |
| `compliance_result` | Compliance result |

---

### terraform-deploy.yml

Deploy pipeline with approval gates.

#### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `working_directory` | string | ✅ | - | Terraform working directory |
| `environment` | string | ✅ | - | GitHub environment name |
| `terraform_version` | string | ❌ | `1.9.0` | Terraform version |
| `aws_region` | string | ❌ | `us-east-1` | AWS region |
| `plan_timeout_minutes` | number | ❌ | `30` | Plan timeout |
| `apply_timeout_minutes` | number | ❌ | `60` | Apply timeout |

#### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AWS_ROLE_ARN` | ✅ | OIDC IAM Role ARN |

---

### terraform-destroy.yml

Destroy pipeline with double approval.

#### Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `working_directory` | string | ✅ | - | Terraform working directory |
| `environment` | string | ✅ | - | GitHub environment name |
| `confirmation_word` | string | ✅ | - | Must match expected |
| `expected_confirmation` | string | ❌ | `DESTROY` | Expected word |
| `terraform_version` | string | ❌ | `1.9.0` | Terraform version |
| `aws_region` | string | ❌ | `us-east-1` | AWS region |
| `destroy_timeout_minutes` | number | ❌ | `60` | Destroy timeout |

## 🔄 Versioning

Use semantic versioning tags to pin workflow versions:

```yaml
# Pin to major version (recommended)
uses: your-org/platform-workflows/.github/workflows/terraform-ci.yml@v1

# Pin to specific version
uses: your-org/platform-workflows/.github/workflows/terraform-ci.yml@v1.2.3

# Pin to branch (not recommended for production)
uses: your-org/platform-workflows/.github/workflows/terraform-ci.yml@main
```

## 🛡️ Security

- All workflows use OIDC for AWS authentication (no long-lived credentials)
- Secrets are passed explicitly, never exposed in logs
- Approval gates enforce human review before changes
- Security scanning is enabled by default

## 📝 License

Apache License 2.0
