# Cloud Security Posture Management (CSPM)

## What is CSPM?

CSPM continuously monitors cloud infrastructure for misconfigurations, compliance violations, and security risks. It provides visibility into your cloud security posture across all accounts and services.

## Common Cloud Misconfigurations

| Misconfiguration | Risk | Prevalence |
|------------------|------|------------|
| Public S3/Blob storage | Data exposure | Very High |
| Overly permissive security groups | Unauthorized access | High |
| Unencrypted data at rest | Data breach | High |
| Missing logging | Undetected attacks | High |
| Default credentials | Account compromise | Medium |
| Unused/orphaned resources | Attack surface | Medium |
| Cross-account access | Lateral movement | Medium |

## Open-Source CSPM Tools

### Prowler (AWS)

```bash
# Install
pip install prowler

# Run full scan
prowler aws

# Scan specific service
prowler aws --service s3 iam ec2

# Scan with specific compliance framework
prowler aws --compliance cis_3.0_aws

# Output formats
prowler aws -M json-ocsf -F prowler-results
prowler aws -M html -F prowler-report
```

### ScoutSuite (Multi-Cloud)

```bash
# Install
pip install scoutsuite

# Scan AWS
scout aws

# Scan Azure
scout azure --cli

# Scan GCP
scout gcp --user-account
```

### CloudSploit

```bash
# Scan AWS
git clone https://github.com/aquasecurity/cloudsploit.git
cd cloudsploit
npm install
node index.js --cloud aws
```

## CIS Benchmarks

The Center for Internet Security (CIS) provides benchmark standards for cloud security.

### Key CIS AWS Checks

```
CIS AWS Foundations Benchmark v3.0:
├── 1. Identity and Access Management
│   ├── 1.1  Maintain current contact details
│   ├── 1.4  Ensure no root access keys exist
│   ├── 1.5  Ensure MFA is enabled for root
│   └── 1.10 Ensure MFA enabled for console access
├── 2. Storage
│   ├── 2.1.1 Ensure S3 Block Public Access
│   └── 2.1.2 Ensure S3 bucket policy denies HTTP
├── 3. Logging
│   ├── 3.1  Ensure CloudTrail is enabled
│   ├── 3.3  Ensure CloudTrail log validation
│   └── 3.7  Ensure VPC Flow Logs enabled
├── 4. Monitoring
│   ├── 4.1-4.15 CloudWatch alarm filters
│   └── (unauthorized API, console sign-in failures, etc.)
└── 5. Networking
    ├── 5.1  Ensure no security groups allow 0.0.0.0/0 to port 22
    └── 5.2  Ensure no security groups allow 0.0.0.0/0 to port 3389
```

## Automated Remediation

### AWS Config Auto-Remediation

```hcl
# Auto-remediate public S3 buckets
resource "aws_config_remediation_configuration" "s3_public" {
  config_rule_name = aws_config_config_rule.s3_public.name

  target_type    = "SSM_DOCUMENT"
  target_id      = "AWS-DisableS3BucketPublicReadWrite"
  automatic      = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60

  parameter {
    name           = "S3BucketName"
    resource_value = "RESOURCE_ID"
  }
}
```

## CSPM Best Practices

1. **Continuous scanning** — not just periodic assessments
2. **Multi-account visibility** — centralize findings from all accounts
3. **Prioritize by risk** — not all misconfigurations are equal
4. **Automated remediation** — fix common issues automatically
5. **Compliance mapping** — map findings to regulatory requirements
6. **Drift detection** — alert when configurations change unexpectedly
7. **Integration** — feed findings into SIEM and ticketing systems
