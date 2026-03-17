# Secret Management

## The Problem with Secrets

Secrets (API keys, passwords, tokens, certificates) are the keys to your infrastructure. Mishandling them is one of the most common and dangerous security issues.

!!! danger "Common Mistakes"
    - Hardcoding secrets in source code
    - Storing secrets in environment variables without protection
    - Sharing secrets over Slack, email, or chat
    - Using the same secret across environments
    - Never rotating secrets

## Secret Detection Tools

### Gitleaks

Scan repositories for hardcoded secrets:

```bash
# Install
brew install gitleaks

# Scan current repository
gitleaks detect --source . --verbose

# Scan in CI pipeline
gitleaks detect --source . --report-format json --report-path gitleaks-report.json

# Protect staged changes (pre-commit)
gitleaks protect --staged --verbose
```

**Custom rules (.gitleaks.toml):**

```toml
[extend]
useDefault = true

[[rules]]
id = "custom-api-key"
description = "Custom API key pattern"
regex = '''(?i)my_service_api_key\s*[:=]\s*['"][a-zA-Z0-9]{32,}['"]'''
tags = ["key", "custom"]

[allowlist]
paths = [
    '''\.gitleaks\.toml$''',
    '''tests/fixtures/''',
]
```

### TruffleHog

Deep secret scanning including git history:

```bash
# Scan git repository (including history)
trufflehog git file://. --only-verified

# Scan GitHub organization
trufflehog github --org=myorg --only-verified

# Scan S3 bucket
trufflehog s3 --bucket=my-bucket --only-verified
```

## Secret Management Solutions

### HashiCorp Vault

Industry-standard secrets management platform.

```bash
# Start Vault (dev mode for learning)
vault server -dev

# Store a secret
vault kv put secret/myapp/config \
    db_password="s3cure_p@ss" \
    api_key="abc123"

# Retrieve a secret
vault kv get secret/myapp/config

# Enable dynamic database credentials
vault secrets enable database
vault write database/config/mydb \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(db:3306)/" \
    allowed_roles="readonly" \
    username="vaultadmin" \
    password="adminpass"
```

**Key Vault Features:**

- Dynamic secrets (generated on demand, auto-expired)
- Encryption as a service (transit secrets engine)
- PKI certificate management
- Identity-based access control
- Audit logging

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name, region="us-east-1"):
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# Usage
db_creds = get_secret("prod/myapp/database")
connection_string = f"postgresql://{db_creds['username']}:{db_creds['password']}@{db_creds['host']}/{db_creds['database']}"
```

### GitHub Actions Secrets

```yaml
# Store secrets in GitHub Settings > Secrets and variables > Actions
# Reference in workflows:
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          API_KEY: ${{ secrets.API_KEY }}
        run: ./deploy.sh
```

!!! warning "GitHub Secrets Best Practices"
    - Use **environment secrets** for environment-specific values
    - Use **organization secrets** for shared values across repos
    - Secrets are **not passed to workflows from forks** (by default)
    - Secrets are **masked in logs** but can still leak through indirect methods

## Secret Rotation

### Why Rotate?

- Limits the blast radius of compromised secrets
- Meets compliance requirements (PCI-DSS, SOC2)
- Reduces risk from insider threats
- Expired access for departed employees

### Rotation Strategy

```
Rotation Schedule:
├── API Keys:           Every 90 days
├── Database Passwords: Every 60 days
├── SSH Keys:           Every 180 days
├── TLS Certificates:   Before expiry (automate with cert-manager)
├── Service Accounts:   Every 90 days
└── Emergency:          Immediately after any suspected breach
```

### Automated Rotation Example

```yaml
# AWS Secrets Manager automatic rotation
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

## Environment-Specific Secret Practices

| Environment | Practice |
|-------------|----------|
| **Local Dev** | Use `.env` files (never commit), or local Vault |
| **CI/CD** | Use platform-native secrets (GitHub Secrets, GitLab CI vars) |
| **Staging** | Use secrets manager with limited-scope credentials |
| **Production** | Use secrets manager with audit logging, rotation, and alerting |

## Twelve-Factor App: Config

Following the [Twelve-Factor App](https://12factor.net/config) methodology:

1. **Never store secrets in code** — use environment or external config
2. **Strict separation of config from code** — same deploy, different config
3. **Config varies between deploys** — code does not
