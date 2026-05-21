# အခန်း (၁၄) - CI/CD Pipeline တွင် Terraform ထည့်သွင်းခြင်း

---

## 14.1 GitOps Workflow for Terraform

### GitOps ဆိုတာ ဘာလဲ?

**GitOps** ဆိုတာ **Git** ကို Infrastructure ၏ **Single Source of Truth** အဖြစ် အသုံးပြုခြင်း ဖြစ်ပါတယ်။

```
Developer ──▶ Git Branch ──▶ Pull Request ──▶ Review ──▶ Merge ──▶ Auto Apply
   │              │              │              │          │          │
   │         Code ပြောင်း    terraform plan   Team       main     terraform
   │                         Auto Comment     Review     branch    apply
   │
   └── terraform code ရေးသား
```

### Branching Strategy

```
main (protected)
 │
 ├── feature/add-rds
 │    └── PR → Plan → Review → Merge → Apply
 │
 ├── feature/update-vpc
 │    └── PR → Plan → Review → Merge → Apply
 │
 └── hotfix/fix-sg-rule
      └── PR → Plan → Review → Merge → Apply
```

---

## 14.2 GitHub Actions ဖြင့် Terraform CI/CD

### Basic CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: 'Terraform CI/CD'

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

env:
  TF_VERSION: '1.7.5'
  AWS_REGION: 'ap-southeast-1'

jobs:
  # ===================================
  # Terraform Validate & Plan
  # ===================================
  terraform-plan:
    name: 'Terraform Plan'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Save Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

      # PR Comment (Plan Result)
      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pushed by: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

  # ===================================
  # Terraform Apply (main branch only)
  # ===================================
  terraform-apply:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

---

## 14.3 Multi-Environment CI/CD

```yaml
# .github/workflows/terraform-multi-env.yml
name: 'Terraform Multi-Environment'

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
    steps:
      - id: set-env
        run: |
          if [[ "${{ github.base_ref || github.ref_name }}" == "main" ]]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          else
            echo "environment=dev" >> $GITHUB_OUTPUT
          fi

  terraform:
    needs: determine-environment
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}

    defaults:
      run:
        working-directory: environments/${{ needs.determine-environment.outputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - run: terraform init
      - run: terraform validate
      - run: terraform plan -no-color

      - name: Apply
        if: github.event_name == 'push'
        run: terraform apply -auto-approve
```

---

## 14.4 Terraform Cloud

```hcl
# Terraform Cloud Backend
terraform {
  cloud {
    organization = "my-organization"

    workspaces {
      name = "myproject-prod"
      # tags = ["app:myproject"]  # Tag-based workspace selection
    }
  }
}
```

---

## 14.5 Environment Variables နှင့် Secrets Management

### GitHub Secrets Configuration

```
GitHub Repository → Settings → Secrets and variables → Actions

Secrets:
├── AWS_ACCESS_KEY_ID
├── AWS_SECRET_ACCESS_KEY
├── TF_VAR_db_password
└── TF_VAR_api_key

Variables:
├── TF_VAR_environment
├── TF_VAR_project_name
└── AWS_REGION
```

### OIDC Authentication (Recommended)

```yaml
# GitHub Actions OIDC - Access Key မလို
- name: Configure AWS Credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
    aws-region: ap-southeast-1
```

```hcl
# AWS IAM Role for GitHub Actions OIDC
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions" {
  name = "github-actions-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:*"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "github_actions" {
  role       = aws_iam_role.github_actions.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
```

---

## 14.6 Policy as Code

### tfsec - Security Scanner

```yaml
# CI Pipeline ထဲ tfsec ထည့်
- name: tfsec
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    soft_fail: true
```

### checkov - Policy Checker

```yaml
- name: Checkov
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: .
    framework: terraform
    soft_fail: true
```

### terraform-docs - Auto Documentation

```yaml
- name: Terraform Docs
  uses: terraform-docs/gh-actions@v1.0.0
  with:
    working-dir: .
    output-file: README.md
    output-method: inject
```

---

## 14.7 လက်တွေ့ - Complete CI/CD Pipeline

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
      - id: terraform_tfsec
```

```bash
# Install
pip install pre-commit
pre-commit install

# Run
pre-commit run --all-files
```

### Makefile

```makefile
# Makefile
.PHONY: init plan apply destroy fmt validate

ENV ?= dev

init:
	cd environments/$(ENV) && terraform init

plan:
	cd environments/$(ENV) && terraform plan

apply:
	cd environments/$(ENV) && terraform apply

destroy:
	cd environments/$(ENV) && terraform destroy

fmt:
	terraform fmt -recursive

validate:
	cd environments/$(ENV) && terraform validate

lint:
	tflint --recursive
	tfsec .

docs:
	terraform-docs markdown . > README.md
```

---

## အခန်း (၁၄) အကျဉ်းချုပ်

- ✅ **GitOps Workflow** - Git as Single Source of Truth
- ✅ **GitHub Actions** CI/CD Pipeline
- ✅ **PR Auto Plan Comment** 
- ✅ **Multi-Environment** CI/CD
- ✅ **Terraform Cloud** Integration
- ✅ **OIDC Authentication** (No Static Keys)
- ✅ **Security Scanning** (tfsec, checkov)
- ✅ **Pre-commit Hooks** နှင့် Makefile

---

*[← အခန်း (၁၃)](chapter-13.md) | [အခန်း (၁၅) - Monitoring & Best Practices →](chapter-15.md)*
