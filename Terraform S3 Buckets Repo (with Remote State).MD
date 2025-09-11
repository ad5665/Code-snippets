# Terraform S3 Buckets Repo (with Remote State)

This repo contains two stages:

* `00-backend-bootstrap/` — creates the S3 bucket and DynamoDB table for **remote state & state locking**.
* `01-infra/` — creates **six** S3 buckets in the same AWS account/tenancy and configures Terraform to use the remote state created in the bootstrap stage.

> ✅ Assumptions: You have AWS credentials configured (e.g., via `AWS_PROFILE` or environment variables) with permissions to manage S3/DynamoDB.

---

## Directory Structure

```
.
├── .gitignore
├── README.md
├── 00-backend-bootstrap
│   ├── main.tf
│   ├── outputs.tf
│   └── variables.tf
└── 01-infra
    ├── backend.hcl           # Fill in values from bootstrap outputs
    ├── main.tf
    ├── outputs.tf
    ├── providers.tf
    └── variables.tf
```

---

## .gitignore

```gitignore
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash logs
crash.log
crash.*.log

# Sensitive overrides
*.auto.tfvars
backend.hcl
```

---

## README.md

````md
# Terraform: 6 S3 Buckets + Remote State

This project uses a two-stage approach:

1. **Bootstrap**: Creates a dedicated S3 bucket for Terraform remote state and a DynamoDB table for state locking.
2. **Infra**: Creates six S3 buckets with best-practice settings (SSE, versioning, public access block, ACLs disabled).

---

## Prereqs
- Terraform >= 1.5
- AWS credentials with permissions for S3 and DynamoDB

## 1) Bootstrap remote state
```bash
cd 00-backend-bootstrap
terraform init
terraform apply -auto-approve
````

Copy the output values into `01-infra/backend.hcl`.

## 2) Deploy infrastructure

```bash
cd ../01-infra
# Edit backend.hcl with values from the bootstrap outputs
terraform init -backend-config=backend.hcl
terraform plan
terraform apply -auto-approve
```

## Customizing bucket names

Edit `01-infra/variables.tf` to change the six bucket names or pass via CLI:

```bash
terraform apply -var='bucket_names=["my-bucket-a","my-bucket-b","..."]'
```

## Clean up

Apply `terraform destroy` in `01-infra` first, then in `00-backend-bootstrap`.

````

---

## 00-backend-bootstrap/main.tf
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.5"
    }
  }
}

provider "aws" {
  region  = var.region
  profile = var.profile
}

# Ensure globally-unique bucket name by appending random suffix
resource "random_string" "suffix" {
  length  = 6
  upper   = false
  special = false
}

locals {
  state_bucket_name = var.state_bucket_base != null && var.state_bucket_base != ""
    ? var.state_bucket_base
    : "tf-remote-state-${random_string.suffix.result}"
}

resource "aws_s3_bucket" "tf_state" {
  bucket = local.state_bucket_name
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_dynamodb_table" "tf_lock" {
  name         = var.dynamodb_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

output "remote_state_bucket" {
  value = aws_s3_bucket.tf_state.bucket
}

output "remote_state_region" {
  value = var.region
}

output "remote_state_dynamodb_table" {
  value = aws_dynamodb_table.tf_lock.name
}
````

---

## 00-backend-bootstrap/variables.tf

```hcl
variable "region" {
  description = "AWS region for the remote state bucket and DynamoDB table"
  type        = string
  default     = "eu-west-1"
}

variable "profile" {
  description = "AWS CLI profile to use (optional)"
  type        = string
  default     = null
}

variable "state_bucket_base" {
  description = "Optional fixed name for the remote state bucket. Leave blank to auto-generate a unique name. Must be globally unique if set."
  type        = string
  default     = null
}

variable "dynamodb_table_name" {
  description = "Name for the DynamoDB state lock table"
  type        = string
  default     = "terraform-state-locks"
}
```

---

## 00-backend-bootstrap/outputs.tf

```hcl
output "instructions" {
  value = "Use these outputs to populate 01-infra/backend.hcl"
}

output "backend_config_example" {
  value = <<EOT
bucket         = \"${aws_s3_bucket.tf_state.bucket}\"
key            = \"global/terraform.tfstate\"
region         = \"${var.region}\"
dynamodb_table = \"${aws_dynamodb_table.tf_lock.name}\"
encrypt        = true
EOT
}
```

---

## 01-infra/backend.hcl

```hcl
# Fill these with values from 00-backend-bootstrap outputs
bucket         = "<remote_state_bucket>"
key            = "global/terraform.tfstate"
region         = "<remote_state_region>"
dynamodb_table = "<remote_state_dynamodb_table>"
encrypt        = true
```

---

## 01-infra/providers.tf

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
  backend "s3" {}
}

provider "aws" {
  region  = var.region
  profile = var.profile
}
```

---

## 01-infra/variables.tf

```hcl
variable "region" {
  description = "AWS region where the buckets will be created"
  type        = string
  default     = "eu-west-1"
}

variable "profile" {
  description = "AWS CLI profile to use (optional)"
  type        = string
  default     = null
}

variable "tags" {
  description = "Common tags applied to all resources"
  type        = map(string)
  default = {
    Project = "s3-buckets"
    Managed = "terraform"
  }
}

variable "bucket_names" {
  description = "List of six S3 bucket names (must be globally unique)."
  type        = list(string)
  default = [
    "cgi-sample-bucket-a",
    "cgi-sample-bucket-b",
    "cgi-sample-bucket-c",
    "cgi-sample-bucket-d",
    "cgi-sample-bucket-e",
    "cgi-sample-bucket-f",
  ]
  validation {
    condition     = length(var.bucket_names) == 6
    error_message = "You must supply exactly six bucket names."
  }
}

variable "enable_versioning" {
  description = "Enable versioning on the buckets"
  type        = bool
  default     = true
}

variable "force_destroy" {
  description = "If true, allows Terraform to delete non-empty buckets (use with caution)"
  type        = bool
  default     = false
}
```

---

## 01-infra/main.tf

```hcl
locals {
  common_tags = var.tags
}

# Create each bucket with best practices
resource "aws_s3_bucket" "buckets" {
  for_each = toset(var.bucket_names)
  bucket   = each.key
  tags     = local.common_tags
  force_destroy = var.force_destroy
}

resource "aws_s3_bucket_ownership_controls" "this" {
  for_each = aws_s3_bucket.buckets
  bucket   = each.value.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

# ACLs are disabled implicitly by BucketOwnerEnforced; define for clarity
resource "aws_s3_bucket_acl" "this" {
  for_each = aws_s3_bucket.buckets
  bucket   = each.value.id
  acl      = "private"
  depends_on = [aws_s3_bucket_ownership_controls.this]
}

resource "aws_s3_bucket_public_access_block" "this" {
  for_each = aws_s3_bucket.buckets
  bucket   = each.value.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  for_each = aws_s3_bucket.buckets
  bucket   = each.value.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  for_each = aws_s3_bucket.buckets
  bucket   = each.value.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}
```

---

## 01-infra/outputs.tf

```hcl
output "created_buckets" {
  description = "All created bucket names"
  value       = keys(aws_s3_bucket.buckets)
}
```
