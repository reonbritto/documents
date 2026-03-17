# Git Security Best Practices

## Repository Security

### Signed Commits

Sign commits to verify author identity and prevent impersonation.

```bash
# Configure GPG key
git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true

# Create a signed commit
git commit -S -m "feat: add authentication module"

# Verify a signed commit
git verify-commit HEAD
```

**SSH Signing (simpler alternative):**

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

### .gitignore Security

Always exclude sensitive files from version control:

```gitignore
# Secrets and credentials
.env
.env.*
*.pem
*.key
*.p12
credentials.json
service-account.json

# IDE and OS files
.idea/
.vscode/settings.json
.DS_Store
Thumbs.db

# Build artifacts
node_modules/
dist/
build/
*.pyc
__pycache__/
```

### Git Hooks for Security

```bash
#!/bin/bash
# .git/hooks/pre-commit - Prevent committing secrets

# Check for common secret patterns
if git diff --cached --diff-filter=ACM | grep -qEi \
  '(password|secret|api_key|token|private_key)\s*[:=]\s*["\x27][^\s]+'; then
  echo "ERROR: Potential secret detected in staged changes!"
  echo "Please remove secrets before committing."
  exit 1
fi

# Run gitleaks
if command -v gitleaks &> /dev/null; then
  gitleaks protect --staged --verbose
  if [ $? -ne 0 ]; then
    echo "ERROR: gitleaks detected secrets in staged changes!"
    exit 1
  fi
fi
```

## Repository Configuration

### Protected Branches

Configure branch protection rules for critical branches:

- Require pull request reviews (minimum 2 reviewers)
- Require status checks to pass (CI/CD, security scans)
- Require signed commits
- Disallow force pushes
- Restrict who can push to the branch
- Require linear history

### CODEOWNERS

Use CODEOWNERS to require reviews from security team for sensitive files:

```
# .github/CODEOWNERS
# Security team reviews infrastructure changes
/terraform/          @org/security-team
/k8s/                @org/security-team
/.github/workflows/  @org/security-team @org/platform-team

# Security reviews for authentication code
/src/auth/           @org/security-team
/src/crypto/         @org/security-team

# Dependency changes need security review
package.json         @org/security-team
package-lock.json    @org/security-team
requirements.txt     @org/security-team
go.sum               @org/security-team
```

## Git History Security

### Removing Sensitive Data from History

If secrets are accidentally committed:

```bash
# Option 1: BFG Repo-Cleaner (recommended)
bfg --replace-text passwords.txt repo.git

# Option 2: git filter-repo
git filter-repo --invert-paths --path secrets.env

# After cleanup: force push (coordinate with team)
git push --force-with-lease --all
```

!!! danger "Important"
    After removing secrets from git history, **always rotate the exposed credentials**. The secrets may have already been compromised.

### Audit Git Access

```bash
# View who has access to the repository
gh api repos/{owner}/{repo}/collaborators --jq '.[].login'

# Audit recent access and actions
gh api repos/{owner}/{repo}/events --jq '.[] | {actor: .actor.login, type: .type, created: .created_at}'
```

## Dependency Management

### Lock Files

Always commit lock files to ensure reproducible builds:

| Package Manager | Lock File |
|----------------|-----------|
| npm | `package-lock.json` |
| yarn | `yarn.lock` |
| pip | `requirements.txt` (pinned) |
| Go | `go.sum` |
| Rust | `Cargo.lock` |

### Automated Dependency Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "security"
    reviewers:
      - "org/security-team"
```
