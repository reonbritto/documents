# Terraform Security

## Common Terraform Security Mistakes

| Mistake | Risk | Fix |
|---------|------|-----|
| Hardcoded secrets | Credential exposure | Use variables + Vault |
| Overly permissive IAM | Privilege escalation | Least privilege policies |
| Public S3 buckets | Data exposure | Block public access by default |
| Unencrypted storage | Data breach | Enable encryption everywhere |
| No state file protection | State tampering | Remote backend with encryption |

## Secure State Management

Terraform state files contain sensitive data — treat them as secrets.

```hcl
# Remote backend with encryption and locking
terraform {
  backend "s3" {
    bucket         = "myorg-terraform-state"
    key            = "prod/infrastructure.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/abc-123"
  }
}
```

**State security checklist:**

- [ ] Remote backend (never local for production)
- [ ] Encryption at rest (S3 + KMS)
- [ ] State locking (DynamoDB)
- [ ] Access control (IAM policies)
- [ ] Versioning enabled (S3 versioning)
- [ ] No secrets in outputs

## Sensitive Variables

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # Prevents display in logs
}

# Never output sensitive values
output "db_connection" {
  value     = "postgresql://admin:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive = true
}
```

**Passing secrets safely:**

```bash
# Via environment variables (preferred in CI)
export TF_VAR_db_password="$(vault kv get -field=password secret/db)"
terraform apply

# Via .tfvars file (never commit!)
# terraform.tfvars is in .gitignore
```

## Secure Resource Patterns

### S3 Bucket

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "myorg-secure-data"
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.data.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_logging" "data" {
  bucket        = aws_s3_bucket.data.id
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "s3-access-logs/"
}
```

### IAM with Least Privilege

```hcl
# Specific permissions, not wildcards
resource "aws_iam_policy" "app_policy" {
  name = "app-limited-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${aws_s3_bucket.data.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      }
    ]
  })
}
```

## Terraform Security Scanning

### Checkov

```bash
# Scan Terraform directory
checkov -d ./terraform

# Scan specific file
checkov -f main.tf

# Output as SARIF
checkov -d ./terraform -o sarif > checkov.sarif

# Skip specific checks
checkov -d ./terraform --skip-check CKV_AWS_18,CKV_AWS_19
```

### tfsec (now part of Trivy)

```bash
# Scan Terraform code
trivy config ./terraform

# With severity filter
trivy config --severity CRITICAL,HIGH ./terraform
```

### Terraform Sentinel (Policy as Code)

```hcl
# Enforce encryption on all S3 buckets
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" and
    rc.mode is "managed" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

main = rule {
    all s3_buckets as _, bucket {
        bucket.change.after.server_side_encryption_configuration is not null
    }
}
```

## Module Security

```hcl
# Pin module versions
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"  # Pin exact version
  # ...
}

# Use private registry for internal modules
module "app" {
  source  = "app.terraform.io/myorg/app-module/aws"
  version = "2.0.0"
}
```
