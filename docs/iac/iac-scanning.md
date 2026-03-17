# IaC Scanning Tools

## Why Scan Infrastructure as Code?

IaC misconfigurations are a leading cause of cloud security breaches. Scanning IaC files catches misconfigurations **before** they're deployed.

```
Common IaC Misconfigurations:
├── Public cloud storage (S3, GCS, Azure Blob)
├── Overly permissive security groups / firewall rules
├── Unencrypted data at rest or in transit
├── Missing logging and monitoring
├── Overly permissive IAM policies
├── Default credentials
└── Missing network segmentation
```

## Tool Comparison

| Tool | Languages | License | Highlights |
|------|-----------|---------|------------|
| **Checkov** | Terraform, CloudFormation, K8s, Dockerfile, Helm | OSS (Apache 2.0) | 1000+ built-in checks, custom policies |
| **Trivy** | Terraform, CloudFormation, K8s, Dockerfile | OSS (Apache 2.0) | All-in-one scanner (images + IaC + code) |
| **tfsec** | Terraform | OSS (MIT) | Now merged into Trivy |
| **KICS** | 15+ platforms | OSS (Apache 2.0) | Wide platform coverage |
| **Terrascan** | Terraform, K8s, Helm, Dockerfile | OSS (Apache 2.0) | OPA-based policies |
| **Snyk IaC** | Terraform, CloudFormation, K8s | Freemium | IDE integration, fix suggestions |

## Checkov Deep Dive

### Basic Usage

```bash
# Install
pip install checkov

# Scan a directory
checkov -d ./terraform

# Scan specific framework
checkov -d ./terraform --framework terraform

# Scan with custom policies
checkov -d ./terraform --external-checks-dir ./custom-policies

# Output formats
checkov -d ./terraform -o json        # JSON output
checkov -d ./terraform -o sarif       # SARIF for GitHub
checkov -d ./terraform -o cli         # Human-readable
```

### Custom Policy (Python)

```python
# custom-policies/s3_versioning.py
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from checkov.common.models.enums import CheckResult, CheckCategories

class S3Versioning(BaseResourceCheck):
    def __init__(self):
        name = "Ensure S3 bucket has versioning enabled"
        id = "CUSTOM_S3_001"
        supported_resources = ["aws_s3_bucket"]
        categories = [CheckCategories.BACKUP_AND_RECOVERY]
        super().__init__(name=name, id=id, categories=categories,
                        supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        versioning = conf.get("versioning", [{}])
        if isinstance(versioning, list) and len(versioning) > 0:
            if versioning[0].get("enabled", [False]) == [True]:
                return CheckResult.PASSED
        return CheckResult.FAILED

check = S3Versioning()
```

### Custom Policy (YAML)

```yaml
# custom-policies/require_encryption.yaml
metadata:
  id: "CUSTOM_ENC_001"
  name: "Ensure RDS instance is encrypted"
  severity: "HIGH"
definition:
  cond_type: "attribute"
  resource_types:
    - "aws_db_instance"
  attribute: "storage_encrypted"
  operator: "is_true"
```

## CI/CD Integration

### GitHub Actions

```yaml
name: IaC Security Scan

on:
  pull_request:
    paths:
      - 'terraform/**'
      - 'k8s/**'
      - 'Dockerfile*'

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Checkov Scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ./terraform
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov.sarif

  trivy-config:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy Config Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: ./terraform
          severity: CRITICAL,HIGH
          exit-code: 1
```

## Baseline and Suppression

### Handling False Positives

```hcl
# Inline suppression in Terraform
resource "aws_s3_bucket" "public_website" {
  #checkov:skip=CKV_AWS_18:This bucket intentionally hosts public website content
  bucket = "my-public-website"
}
```

```yaml
# .checkov.yml - global configuration
skip-check:
  - CKV_AWS_18  # Public S3 - handled by CDN policy
  - CKV_AWS_145 # RDS encryption - using application-level encryption
check:
  - CKV_AWS_20
  - CKV_AWS_57
framework:
  - terraform
compact: true
```

## Best Practices

1. **Scan early** — IDE plugins for real-time feedback
2. **Scan in CI** — Block PRs with critical misconfigurations
3. **Custom policies** — Enforce organization-specific standards
4. **Baseline management** — Track and reduce suppressions over time
5. **Multiple tools** — Different tools catch different issues
6. **Drift detection** — Compare deployed state against IaC
