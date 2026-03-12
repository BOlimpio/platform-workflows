# platform-workflows

> **Role:** Centralized, reusable GitHub Actions workflow library for Terraform infrastructure management.

This repository provides three production-grade, reusable workflows consumed by all Terraform projects via GitHub's `workflow_call` mechanism. Changes made here propagate automatically to every consumer.

---

## Table of Contents

- [Why Reusable Workflows](#why-reusable-workflows)
- [Workflow Overview](#workflow-overview)
- [How to Consume These Workflows](#how-to-consume-these-workflows)
- [terraform-ci.yml — CI Pipeline](#terraform-ciyml--ci-pipeline)
- [terraform-deploy.yml — Deploy Pipeline](#terraform-deployyml--deploy-pipeline)
- [terraform-destroy.yml — Destroy Pipeline](#terraform-destroyyml--destroy-pipeline)
- [Security Model](#security-model)
- [Versioning Strategy](#versioning-strategy)
- [Learning Topics](#learning-topics)

---

## Why Reusable Workflows

Traditional approaches copy-paste workflow YAML into every repo. This creates drift: repos diverge, security fixes aren't backported, and every team reinvents the same CI stages.

**Reusable workflows solve this:**

```
platform-workflows (single source of truth)
        │
        ├── terraform-ci.yml      ◄── consumed by platform-playground
        ├── terraform-deploy.yml  ◄── consumed by platform-playground
        └── terraform-destroy.yml ◄── consumed by platform-playground
                                        and any future Terraform project
```

| Benefit | Description |
|---|---|
| **DRY** | Define CI stages once; all consumers inherit them |
| **Centralized security** | Add a Checkov check or OIDC requirement once — it applies everywhere |
| **Consistent standards** | Format, lint, test, scan — same pipeline for every repo |
| **Easy updates** | Bump the workflow tag in consumers to adopt improvements |
| **Separation of concerns** | Application teams focus on Terraform code; platform team owns CI/CD |

---

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  terraform-ci.yml                                               │
│                                                                 │
│  format → validate → lint → test → security → compliance       │
│                          ↓           ↓            ↓            │
│                      (parallel)  (parallel)   (parallel)       │
│                                                                 │
│  Modes:  • environment (test root module)                       │
│          • modules (test individual modules via matrix)         │
└─────────────────────────────────────────────────────────────────┘
                    consumed by CI/CD and PR pipelines

┌─────────────────────────────────────────────────────────────────┐
│  terraform-deploy.yml                                           │
│                                                                 │
│  plan → [approval gate] → apply                                 │
│                                                                 │
│  • Plan artifact signed and stored                              │
│  • Apply uses exactly the same plan (no drift)                  │
│  • Approval gate uses GitHub Environments                       │
└─────────────────────────────────────────────────────────────────┘
                    consumed by deploy action

┌─────────────────────────────────────────────────────────────────┐
│  terraform-destroy.yml                                          │
│                                                                 │
│  validate-word → [approval-1] → plan-destroy → [approval-2]    │
│                                                          ↓      │
│                                                       destroy   │
│                                                                 │
│  • Requires typing "DESTROY" as confirmation word               │
│  • Double approval gate before irreversible action              │
└─────────────────────────────────────────────────────────────────┘
                    consumed by destroy action
```

---

## How to Consume These Workflows

In any Terraform repository, call these workflows using `uses:`:

```yaml
# .github/workflows/ci-cd.yml (in consumer repo)

jobs:
  ci:
    uses: YOUR_ORG/platform-workflows/.github/workflows/terraform-ci.yml@v1
    with:
      working_directory: environments/dev
      environment: dev
      terraform_version: "1.10.0"
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}

  deploy:
    needs: ci
    if: github.event_name == 'workflow_dispatch'
    uses: YOUR_ORG/platform-workflows/.github/workflows/terraform-deploy.yml@v1
    with:
      working_directory: environments/dev
      environment: dev
      terraform_version: "1.10.0"
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

**Tag reference:** `@v1` uses the floating `v1` tag (always points to latest v1.x). Pin to a specific commit SHA for maximum stability.

---

## terraform-ci.yml — CI Pipeline

### What It Does

Runs a full quality gate pipeline in 7 sequential/parallel stages. Designed to fail fast: format errors stop the pipeline before slower stages run.

```
Stage 1: format      (fast — ~10s)  — terraform fmt check
Stage 2: validate    (fast — ~20s)  — terraform validate + provider init
Stage 3: lint        (medium — ~30s) — tflint with AWS ruleset
Stage 3: test        (medium — ~30s) — terraform test (plan mode, optional apply)
Stage 4: security    (medium — ~30s) — checkov SARIF scan
Stage 4: compliance  (medium — ~45s) — terraform-compliance (Gherkin rules)
Stage 5: cost        (fast — ~15s)  — infracost PR comment (PRs only)
Stage 6: summary     (fast — ~5s)   — combined results table
```

### Test Modes

The CI pipeline supports two distinct test modes, controlled by `test_mode`:

**`environment` mode** (default)
- Tests the root module (e.g., `environments/dev`)
- Runs `terraform test -filter=tests/plan.tftest.hcl`
- Optionally runs apply tests if `run_apply_tests=true`

**`modules` mode**
- Automatically discovers all modules under `modules_base_path` (e.g., `modules/aws/`)
- Excludes paths matching `exclude_modules_pattern` (default: `issue-*`)
- Runs each module's `tests/plan.tftest.hcl` in parallel via a GitHub Actions matrix
- Each module gets its own job with isolated AWS credentials

```yaml
# Example: test all modules
with:
  test_mode: modules
  modules_to_test: all     # or "s3,lambda" for specific modules

# Example: test single module
with:
  test_mode: modules
  modules_to_test: s3
```

### Compliance Approach

The compliance stage uses `terraform-compliance` with [Gherkin](https://cucumber.io/docs/gherkin/) feature files:

```gherkin
# compliance/features/encryption.feature
Feature: Resources must have encryption enabled
  Scenario: S3 buckets must have encryption
    Given I have aws_s3_bucket_server_side_encryption_configuration defined
    Then it must contain rule
```

Compliance runs a **mock plan** (no real AWS credentials needed):
- Overrides backend to `local`
- Injects a mock AWS provider with `skip_credentials_validation = true`
- Runs `terraform plan -refresh=false` to generate `tfplan.json`
- Passes `tfplan.json` to `terraform-compliance` for rule evaluation

This means compliance checks run even without AWS connectivity — fast and cost-free.

### Inputs Reference

| Input | Type | Default | Description |
|---|---|---|---|
| `working_directory` | string | required | Path to the Terraform root module (e.g., `environments/dev`) |
| `environment` | string | required | Environment name (dev, staging, production) |
| `terraform_version` | string | `"1.9.0"` | Terraform version to install |
| `tflint_version` | string | `"v0.50.0"` | TFLint version to install |
| `python_version` | string | `"3.11"` | Python version for compliance tools |
| `aws_region` | string | `"us-east-1"` | AWS region for provider configuration |
| `test_mode` | string | `"environment"` | Test strategy: `environment` or `modules` |
| `modules_to_test` | string | `"all"` | Comma-separated module names or `"all"` |
| `modules_base_path` | string | `"modules/aws"` | Base path to scan for modules |
| `test_directory` | string | `"tests"` | Subdirectory containing test files |
| `plan_test_file` | string | `"plan.tftest.hcl"` | Plan test filename (must exist in `test_directory`) |
| `apply_test_file` | string | `"apply.tftest.hcl"` | Apply test filename |
| `run_apply_tests` | bool | `false` | Run apply tests (creates real AWS resources — costs money) |
| `enable_security_scan` | bool | `true` | Run Checkov security scan |
| `enable_compliance` | bool | `true` | Run terraform-compliance |
| `enable_cost_estimation` | bool | `true` | Run Infracost (PRs only) |
| `enable_lint` | bool | `true` | Run TFLint |
| `lint_child_modules` | bool | `false` | Also lint modules referenced from root |
| `exclude_modules_pattern` | string | `"issue-*"` | Glob pattern for modules to skip in discovery |
| `compliance_features_path` | string | `"compliance/features"` | Path to Gherkin `.feature` files |
| `compliance_tfvars_file` | string | `""` | Path to a `.tfvars` file for the compliance plan (relative to `working_directory`) |
| `checkov_config_file` | string | `".checkov.yaml"` | Path to Checkov configuration |
| `tflint_config_file` | string | `".tflint.hcl"` | Path to TFLint configuration |

### Secrets Reference

| Secret | Required | Description |
|---|---|---|
| `AWS_ROLE_ARN` | yes | IAM Role ARN for OIDC authentication |
| `INFRACOST_API_KEY` | no | Infracost API key for cost estimation |

### Outputs Reference

| Output | Description |
|---|---|
| `format_result` | Result of the format check job (`success`, `failure`, `skipped`) |
| `validate_result` | Result of the validate job |
| `lint_result` | Result of the lint job |
| `test_result` | Result of the test job |
| `security_result` | Result of the security scan job |
| `compliance_result` | Result of the compliance job |

---

## terraform-deploy.yml — Deploy Pipeline

### What It Does

Manages the full lifecycle of a Terraform apply in a safe, auditable way:

```
┌──────────┐     ┌──────────────────┐     ┌───────────┐
│   Plan   │────►│  Approval Gate   │────►│   Apply   │
│          │     │                  │     │           │
│ • init   │     │ GitHub Env gate  │     │ downloads │
│ • plan   │     │ reviewer approves│     │ artifact  │
│ • upload │     │ (or auto for dev)│     │ applies   │
│ artifact │     │                  │     │ same plan │
└──────────┘     └──────────────────┘     └───────────┘
```

**Key design decisions:**

- **Plan before approval** — reviewers see exactly what will change before approving
- **Artifact integrity** — the plan binary is stored as a GitHub Actions artifact; apply downloads and uses *that exact plan*, preventing drift between what was reviewed and what is applied
- **Environment-based approval** — uses GitHub Environments so `dev` can auto-approve while `staging` and `production` require human review
- **No redundant plan** — apply does NOT re-run `terraform plan`; it uses `terraform apply tfplan` with the stored artifact

### Inputs Reference

| Input | Type | Default | Description |
|---|---|---|---|
| `working_directory` | string | required | Path to the Terraform root module |
| `environment` | string | required | GitHub Environment name (controls approval gate) |
| `terraform_version` | string | `"1.9.0"` | Terraform version |
| `aws_region` | string | `"us-east-1"` | AWS region |
| `plan_timeout_minutes` | number | `30` | Timeout for the plan job |
| `apply_timeout_minutes` | number | `60` | Timeout for the apply job |

### Outputs Reference

| Output | Description |
|---|---|
| `plan_exitcode` | Terraform plan exit code: `0` = no changes, `2` = changes to apply |
| `apply_result` | Apply job result (`success`, `failure`, `skipped`) |

---

## terraform-destroy.yml — Destroy Pipeline

### What It Does

Provides a multi-layer safety system for irreversible infrastructure destruction:

```
┌──────────────┐  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────┐
│   Validate   │─►│  Approval 1 │─►│ Plan Destroy │─►│  Approval 2 │─►│  Destroy │
│              │  │             │  │              │  │             │  │          │
│ Check "DESTROY" │ First gate  │  │ Show exactly │  │ Final gate  │  │ terraform│
│ word matches │  │             │  │ what's lost  │  │ after seeing│  │ apply    │
│              │  │             │  │              │  │ the plan    │  │ -destroy │
└──────────────┘  └─────────────┘  └──────────────┘  └─────────────┘  └──────────┘
     Layer 1           Layer 2           Layer 3           Layer 4        Action
```

**Safety layers:**

1. **Confirmation word** — must type `DESTROY` (uppercase) to proceed
2. **First approval gate** — human reviewer must approve before planning
3. **Plan review** — destroy plan shows exactly which resources will be deleted
4. **Second approval gate** — reviewer confirms after seeing the plan
5. **Production warning** — Step Summary shows explicit warning when destroying production

### Inputs Reference

| Input | Type | Default | Description |
|---|---|---|---|
| `working_directory` | string | required | Path to the Terraform root module |
| `environment` | string | required | GitHub Environment name |
| `confirmation_word` | string | required | Must equal `expected_confirmation` to proceed |
| `expected_confirmation` | string | `"DESTROY"` | The required confirmation word |
| `terraform_version` | string | `"1.9.0"` | Terraform version |
| `aws_region` | string | `"us-east-1"` | AWS region |
| `destroy_timeout_minutes` | number | `60` | Timeout for the destroy job |

### Outputs Reference

| Output | Description |
|---|---|
| `destroy_result` | Destroy job result (`success`, `failure`, `skipped`) |

---

## Security Model

### OIDC Authentication (No Static Credentials)

Every job that needs AWS access uses GitHub's OIDC provider:

```yaml
permissions:
  id-token: write    # required for OIDC token request
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      role-session-name: terraform-ci-${{ github.run_id }}
      aws-region: us-east-1
```

**How OIDC works:**
1. GitHub runner requests a short-lived OIDC token (JWT) from GitHub's token service
2. Token contains claims: `repo`, `ref`, `environment`, `job_workflow_ref`, etc.
3. AWS STS exchanges the JWT for temporary credentials (`AssumeRoleWithWebIdentity`)
4. Credentials last only for the job duration (~1 hour max)
5. No secrets stored anywhere — `AWS_ROLE_ARN` is not sensitive (it's just an ARN)

### Checkov Security Scanning

Checkov scans run on every CI execution and upload results as SARIF to GitHub Security → Code Scanning. Findings appear in the repository's Security tab.

Skip rules are documented with justification in `.checkov.yaml`:
```yaml
skip-check:
  - CKV_AWS_117    # Lambda inside VPC — intentionally omitted (cost/simplicity)
  - CKV_AWS_158    # CloudWatch KMS — using default SSE
  # ... each skip has a documented reason
```

### Plan Artifact Integrity

The deploy workflow ensures **what was reviewed = what was applied**:
```
plan job  ──► uploads artifact "tfplan-{environment}"
                         │
apply job ──► downloads artifact "tfplan-{environment}" ──► terraform apply tfplan
```

If the artifact upload fails, apply cannot proceed. This prevents the "I approved a plan but something different was applied" scenario.

---

## Versioning Strategy

Workflows are tagged with semantic versions:

```bash
# Create a new release
git tag -a v1.2.0 -m "Add compliance_tfvars_file input"
git push origin v1.2.0

# Update floating tag
git tag -fa v1 -m "Update v1 to v1.2.0"
git push origin v1 --force
```

Consumers use the floating tag for automatic updates:
```yaml
uses: YOUR_ORG/platform-workflows/.github/workflows/terraform-ci.yml@v1
```

Or pin to a specific version for controlled updates:
```yaml
uses: YOUR_ORG/platform-workflows/.github/workflows/terraform-ci.yml@v1.2.0
```

---

## Learning Topics

This repository demonstrates the following DevOps and GitHub Actions concepts:

| Topic | Where to Look |
|---|---|
| **Reusable workflows** (`workflow_call`) | All three workflow files — `on: workflow_call:` section |
| **Workflow inputs and outputs** | `inputs:`, `outputs:` blocks in each workflow |
| **Passing secrets to reusable workflows** | `secrets:` block in `workflow_call` |
| **GitHub Environments as approval gates** | `environment:` on job in `terraform-deploy.yml` |
| **Double approval gate** | `terraform-destroy.yml` — two separate `environment:` jobs |
| **Dynamic matrix from outputs** | `discover-modules` job feeding `test-modules` matrix |
| **OIDC authentication** | `configure-aws-credentials` step in every AWS job |
| **Artifact upload/download** | Plan artifact in `terraform-deploy.yml` |
| **Job dependencies** (`needs:`) | Sequential stages with parallel sub-stages |
| **Conditional jobs** (`if:`) | `enable_lint`, `enable_compliance`, `run_apply_tests` |
| **SARIF security upload** | `security` job uploading Checkov results |
| **Cost estimation in PRs** | `cost` job with Infracost PR comment |
| **Compliance as code** | Gherkin feature files + terraform-compliance |
| **Concurrency control** | Consumer workflow `concurrency:` group by environment |
| **Step Summaries** | `$GITHUB_STEP_SUMMARY` writes in multiple jobs |
| **Plugin caching** | Terraform provider cache with `actions/cache@v4` |
| **Terraform native testing** | `terraform test` in plan and apply modes |
| **Mock provider for compliance** | Local backend + skipped validation in compliance job |
