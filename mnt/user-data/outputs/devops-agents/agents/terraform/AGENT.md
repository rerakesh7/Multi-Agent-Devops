# 🏗️ Agent-006: Terraform Agents by Environment
# Provider versioning · Multi-region · Smart state · DB protection
# HITL: ALWAYS required for terraform apply in prod

```
╔══════════════════════════════════════════════════════════════════╗
║           TERRAFORM MULTI-ENV AGENT v2.0                         ║
║   Environment-Aware · State Isolation · DB Protection            ║
║   ⚠️  PRODUCTION APPLY REQUIRES 2-PERSON APPROVAL                 ║
╚══════════════════════════════════════════════════════════════════╝

TERRAFORM WORKSPACE ARCHITECTURE:

  ┌─────────────────────────────────────────────────────────────┐
  │                    STATE FILE STRATEGY                       │
  │                                                              │
  │  terraform/                                                  │
  │  ├── environments/                                          │
  │  │   ├── dev/          ◀── State: GCS/S3/AzureBlob (dev)   │
  │  │   ├── staging/      ◀── State: GCS/S3/AzureBlob (stg)   │
  │  │   └── prod/         ◀── State: GCS/S3/AzureBlob (prd)   │
  │  │       ├── main.tf                                        │
  │  │       ├── variables.tf                                   │
  │  │       └── terraform.tfvars                               │
  │  ├── modules/                                               │
  │  │   ├── gke-cluster/                                       │
  │  │   ├── eks-cluster/                                       │
  │  │   ├── aks-cluster/                                       │
  │  │   ├── database/                                          │
  │  │   └── networking/                                        │
  │  └── shared/          ◀── Shared state (DNS, certs, etc.)  │
  └─────────────────────────────────────────────────────────────┘
```

---

## Skill: Provider Versioning (All 3 Clouds)

### `terraform/shared/versions.tf`
```hcl
terraform {
  # Pin Terraform version
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    # ── GCP ──────────────────────────────────────────────
    google = {
      source  = "hashicorp/google"
      version = "~> 6.12.0"    # Minor updates OK, no major
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.12.0"
    }

    # ── AWS ──────────────────────────────────────────────
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.81.0"
    }

    # ── Azure ────────────────────────────────────────────
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.14.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 3.0.2"
    }

    # ── Kubernetes ────────────────────────────────────────
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.35.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.17.0"
    }

    # ── Misc ──────────────────────────────────────────────
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6.0"
    }
  }
}
```

---

## Skill: Smart State File Strategy

### GCP State Backend
```hcl
# terraform/environments/prod/backend.tf
terraform {
  backend "gcs" {
    bucket = "my-company-tfstate-prod"    # Separate bucket per env
    prefix = "gcp/prod/gke-us-central1"  # Unique prefix per stack
    
    # Encryption with CMEK
    encryption_key = "projects/my-project/locations/us-central1/keyRings/terraform/cryptoKeys/tfstate"
  }
}

# State bucket configuration (one-time setup)
resource "google_storage_bucket" "tfstate" {
  name     = "my-company-tfstate-prod"
  location = "US"
  
  versioning {
    enabled = true    # 90-day version history
  }
  
  lifecycle_rule {
    condition {
      num_newer_versions = 20
      with_state        = "ARCHIVED"
    }
    action { type = "Delete" }
  }
  
  # Prevent accidental deletion
  prevent_destroy = true
  
  uniform_bucket_level_access = true
}
```

### AWS State Backend
```hcl
# S3 + DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-company-tfstate-prod"
    key            = "aws/prod/eks-us-east-1/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/mrk-abc123"
    dynamodb_table = "terraform-state-lock"  # Prevents concurrent applies
  }
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "my-company-tfstate-prod"
  
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_dynamodb_table" "tfstate_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  lifecycle { prevent_destroy = true }
}
```

### Azure State Backend
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate-prod"
    storage_account_name = "mycompanytfstateprod"
    container_name       = "tfstate"
    key                  = "aks/prod/eastus/terraform.tfstate"
    use_oidc             = true    # OIDC auth — no static keys
  }
}
```

---

## Skill: Multi-Region Infrastructure

### GCP Multi-Region GKE
```hcl
# modules/gke-cluster/main.tf
module "gke_us_central1" {
  source = "../../modules/gke-cluster"
  
  project_id = var.project_id
  region     = "us-central1"
  name       = "prod-us-c1"
  
  # Regional cluster (3 zones for HA)
  regional = true
  zones    = ["us-central1-a", "us-central1-b", "us-central1-c"]
  
  node_pools = [{
    name               = "default-pool"
    machine_type       = "e2-standard-4"
    min_count          = 3
    max_count          = 20
    disk_size_gb       = 100
    disk_type          = "pd-ssd"
    auto_repair        = true
    auto_upgrade       = true
    initial_node_count = 3
    preemptible        = false    # No preemptible in prod!
  }]
  
  # Security
  enable_binary_authorization  = true
  enable_workload_identity     = true
  enable_shielded_nodes        = true
  master_authorized_networks   = var.allowed_networks
}

module "gke_us_east1" {
  source = "../../modules/gke-cluster"
  # Mirror config, different region
  region = "us-east1"
  name   = "prod-us-e1"
  # ... same config
}

# Global load balancer routing traffic between regions
resource "google_compute_global_forwarding_rule" "multi_region" {
  name       = "prod-global-lb"
  target     = google_compute_target_https_proxy.main.id
  port_range = "443"
  ip_address = google_compute_global_address.main.address
}
```

---

## Skill: Prevent DB Downtime

```hcl
# modules/database/main.tf — DB module with downtime prevention
resource "google_sql_database_instance" "prod" {
  name             = "prod-postgres-${var.region}"
  database_version = "POSTGRES_16"
  region           = var.region
  
  deletion_protection = true    # ⚠️ PREVENTS terraform destroy!
  
  settings {
    tier              = "db-custom-8-32768"
    availability_type = "REGIONAL"     # Multi-AZ (HA)
    
    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"    # 3AM UTC
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }
    
    maintenance_window {
      day          = 7    # Sunday
      hour         = 4    # 4AM UTC
      update_track = "stable"
    }
    
    database_flags {
      name  = "max_connections"
      value = "200"
    }
    
    insights_config {
      query_insights_enabled  = true
      record_application_tags = true
    }
  }
  
  lifecycle {
    prevent_destroy = true
    
    # Prevent accidental tier/region changes that cause downtime
    ignore_changes = [
      settings[0].disk_size,    # Allow auto-growth
    ]
    
    # These changes WILL cause downtime — require explicit approval
    # (agent will create HITL gate for these):
    # - database_version (major version upgrade)
    # - region
    # - availability_type (ZONAL ↔ REGIONAL)
  }
}

# ── DB Downtime Prevention Checklist (HITL) ──────────────
# Agent generates this before ANY DB terraform changes:
#
# □ Is this change destructive? (check: terraform plan | grep -i destroy)
# □ Will this cause downtime? (check: maintenance window set correctly)
# □ Is backup recent? (check: last backup < 24h ago)
# □ Has migration been tested on staging?
# □ Is rollback plan documented?
# □ Is on-call engineer aware?
#
# For DB: require 2-person approval minimum
```

### DB Schema Migration Safety
```hcl
# Use Flyway/Liquibase via null_resource for tracked migrations
resource "null_resource" "db_migration" {
  # Only run when migration files change
  triggers = {
    migrations_hash = sha256(join(",", [for f in fileset("${path.module}/migrations", "*.sql") : f]))
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "Running migrations on ${var.environment}"
      echo "═══════════════════════════════════"
      echo "⚠️  HUMAN APPROVAL REQUIRED for prod migrations"
      
      # Dry run first
      flyway -url=${var.db_url} info
      
      # Show pending migrations
      PENDING=$(flyway -url=${var.db_url} info | grep "Pending" | wc -l)
      echo "Pending migrations: $PENDING"
      
      # Apply (only after HITL approval via GitHub Environment)
      flyway -url=${var.db_url} migrate
    EOT
  }
}
```

---

## Terraform CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  push:
    paths: ['terraform/**']
  pull_request:
    paths: ['terraform/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
    
    - uses: hashicorp/setup-terraform@b9cd54531c595477b8a08758c76d81535a5b65e6  # v3.1.2
      with:
        terraform_version: "~1.9.0"
        terraform_wrapper: false
    
    - name: Terraform Format Check
      run: terraform fmt -check -recursive terraform/
    
    - name: Validate all environments
      run: |
        for env in dev staging prod; do
          echo "Validating $env..."
          cd terraform/environments/$env
          terraform init -backend=false
          terraform validate
          cd -
        done
    
    - name: tflint
      uses: terraform-linters/setup-tflint@19a52fbac37dacb22a09518e4ef6ee234f2d4987  # v4.0.0
      with: { tflint_version: latest }
    
    - name: tfsec security scan
      run: |
        docker run --rm -v $(pwd):/src aquasec/tfsec:latest /src/terraform \
          --minimum-severity HIGH

  plan:
    needs: validate
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    environment: ${{ matrix.environment }}
    steps:
    - name: Terraform Plan
      id: plan
      run: |
        cd terraform/environments/${{ matrix.environment }}
        terraform init
        terraform plan -out=tfplan-${{ matrix.environment }} -no-color 2>&1 | tee plan.txt
        
        # Check for dangerous operations
        if grep -E "will be destroyed|forces replacement" plan.txt; then
          echo "⚠️  DESTRUCTIVE OPERATIONS DETECTED"
          echo "::warning::Destructive changes in plan for ${{ matrix.environment }}"
        fi
    
    - name: Post plan to PR
      uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea  # v7.0.1
      with:
        script: |
          const plan = require('fs').readFileSync('plan.txt', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `### Terraform Plan: \`${{ matrix.environment }}\`\n\`\`\`\n${plan.slice(0,65000)}\n\`\`\``
          })

  apply-prod:
    needs: plan
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://prod.example.com
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
    - name: Terraform Apply (PRODUCTION)
      # Requires GitHub Environment approval (2 reviewers configured)
      run: |
        cd terraform/environments/prod
        terraform apply tfplan-prod -auto-approve
        echo "✅ Production infrastructure updated"
```
