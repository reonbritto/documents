# Image Scanning & Signing

## Why Scan Container Images?

Container images can contain:

- Known vulnerabilities (CVEs) in OS packages
- Vulnerable application dependencies
- Misconfigurations (running as root, exposed secrets)
- Malware or backdoors in untrusted base images

## Scanning Tools

### Trivy

The most popular open-source scanner — covers vulnerabilities, misconfigurations, secrets, and licenses.

```bash
# Scan a container image
trivy image myapp:v1.0.0

# Scan with severity filter
trivy image --severity CRITICAL,HIGH myapp:v1.0.0

# Output as SARIF (for GitHub Security tab)
trivy image --format sarif --output trivy.sarif myapp:v1.0.0

# Scan filesystem (for IaC and code)
trivy fs --security-checks vuln,secret,config .

# Scan Kubernetes cluster
trivy k8s --report summary cluster
```

**CI/CD Integration:**

```yaml
- name: Trivy Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

### Grype

```bash
# Scan image
grype myapp:v1.0.0

# Scan from SBOM
grype sbom:sbom.json

# Only show fixable vulnerabilities
grype myapp:v1.0.0 --only-fixed
```

### Docker Scout

```bash
# Native Docker scanning
docker scout cves myapp:v1.0.0

# Compare images
docker scout compare myapp:v1.0.0 --to myapp:v0.9.0

# Recommendations
docker scout recommendations myapp:v1.0.0
```

## Software Bill of Materials (SBOM)

An SBOM is a complete inventory of all components in your software.

### Generating SBOMs

```bash
# Syft - generate SBOM
syft myapp:v1.0.0 -o spdx-json > sbom.spdx.json
syft myapp:v1.0.0 -o cyclonedx-json > sbom.cdx.json

# Trivy SBOM
trivy image --format cyclonedx --output sbom.json myapp:v1.0.0
```

### SBOM Formats

| Format | Standard | Use Case |
|--------|----------|----------|
| **SPDX** | ISO/IEC 5962:2021 | License compliance, government |
| **CycloneDX** | OWASP | Security, vulnerability management |
| **Syft JSON** | Anchore | Integration with Grype |

## Image Signing

### Cosign (Sigstore)

```bash
# Generate key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry/myapp:v1.0.0

# Verify signature
cosign verify --key cosign.pub myregistry/myapp:v1.0.0

# Keyless signing (using OIDC identity)
cosign sign myregistry/myapp:v1.0.0
# Opens browser for OIDC authentication

# Verify keyless signature
cosign verify \
  --certificate-identity=developer@company.com \
  --certificate-oidc-issuer=https://accounts.google.com \
  myregistry/myapp:v1.0.0
```

### Attach Attestations

```bash
# Attach SBOM
cosign attest --key cosign.key \
  --predicate sbom.spdx.json \
  --type spdxjson \
  myregistry/myapp:v1.0.0

# Attach vulnerability scan results
cosign attest --key cosign.key \
  --predicate trivy-results.json \
  --type vuln \
  myregistry/myapp:v1.0.0

# Verify attestation
cosign verify-attestation --key cosign.pub \
  --type spdxjson \
  myregistry/myapp:v1.0.0
```

## Registry Security

### Private Registry Best Practices

1. **Enable authentication** — no anonymous pulls
2. **Use TLS** — encrypt all traffic
3. **Enable content trust** — Docker Content Trust (DCT)
4. **Implement vulnerability scanning** — scan on push
5. **Set retention policies** — clean up old/unused images
6. **Enable audit logging** — track who pulls/pushes what

### Admission Control

Enforce that only scanned and signed images can be deployed:

```yaml
# Kyverno: Require image signature
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "myregistry/*"
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

## Scanning Best Practices

| Practice | Description |
|----------|-------------|
| Scan on build | Every CI build scans the image |
| Scan on push | Registry scans on image push |
| Continuous scanning | Re-scan deployed images for new CVEs |
| Block critical CVEs | Fail pipelines on critical findings |
| Track SBOM | Generate and store SBOM with every image |
| Sign everything | Sign images and attestations |
| Verify on deploy | Admission controller verifies signatures |
