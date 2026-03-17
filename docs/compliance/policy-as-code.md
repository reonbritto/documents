# Policy as Code

## What is Policy as Code?

Policy as Code (PaC) means expressing security and compliance policies as machine-readable code that can be version-controlled, tested, and automatically enforced.

```
Traditional Compliance:                 Policy as Code:
├── PDF policy documents             ├── Version-controlled policies
├── Manual periodic audits           ├── Automated continuous checks
├── Human interpretation             ├── Machine enforcement
├── Inconsistent application         ├── Consistent enforcement
└── Annual evidence gathering        └── Continuous evidence
```

## Open Policy Agent (OPA)

OPA is the most popular open-source policy engine, using the Rego language.

### Basic Rego Policy

```rego
# policies/deployment.rego
package deployment

# Default deny
default allow = false

# Allow deployment if all conditions pass
allow {
    input.image_signed
    input.vulnerability_scan_passed
    input.sast_scan_passed
    not high_severity_cve
    valid_registry
}

# Check for high severity CVEs
high_severity_cve {
    some vuln
    input.vulnerabilities[vuln].severity == "CRITICAL"
}

# Require approved registries
valid_registry {
    allowed_registries := {
        "myregistry.azurecr.io",
        "123456789012.dkr.ecr.us-east-1.amazonaws.com"
    }
    some registry in allowed_registries
    startswith(input.image, registry)
}
```

### Testing OPA Policies

```rego
# policies/deployment_test.rego
package deployment

# Test: Allow signed, scanned image
test_allow_valid_deployment {
    allow with input as {
        "image": "myregistry.azurecr.io/myapp:v1.0.0",
        "image_signed": true,
        "vulnerability_scan_passed": true,
        "sast_scan_passed": true,
        "vulnerabilities": []
    }
}

# Test: Deny unsigned image
test_deny_unsigned_image {
    not allow with input as {
        "image": "myregistry.azurecr.io/myapp:v1.0.0",
        "image_signed": false,
        "vulnerability_scan_passed": true,
        "sast_scan_passed": true,
        "vulnerabilities": []
    }
}

# Test: Deny critical CVE
test_deny_critical_cve {
    not allow with input as {
        "image": "myregistry.azurecr.io/myapp:v1.0.0",
        "image_signed": true,
        "vulnerability_scan_passed": false,
        "sast_scan_passed": true,
        "vulnerabilities": [{"id": "CVE-2024-1234", "severity": "CRITICAL"}]
    }
}
```

```bash
# Run OPA tests
opa test policies/ -v
```

## Kubernetes Policy Enforcement

### Kyverno

```yaml
# Require non-root containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-user
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-root-user
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Containers must not run as root"
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
            containers:
              - securityContext:
                  runAsNonRoot: true
                  allowPrivilegeEscalation: false

---
# Enforce image registry
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allowed-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must come from approved registries"
        pattern:
          spec:
            containers:
              - image: "myregistry.io/* | 123456789012.dkr.ecr.us-east-1.amazonaws.com/*"
```

### OPA/Gatekeeper

```yaml
# ConstraintTemplate for allowed registries
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
      validation:
        openAPIV3Schema:
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedregistries
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with_allowed(container.image)
          msg := sprintf("Image '%v' is not from an allowed registry", [container.image])
        }
        starts_with_allowed(image) {
          allowed := input.parameters.registries[_]
          startswith(image, allowed)
        }

---
# Apply the constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    registries:
      - "myregistry.io/"
      - "123456789012.dkr.ecr.us-east-1.amazonaws.com/"
```

## Cloud Policy Enforcement

### AWS Service Control Policies

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireEncryptionAtRest",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
        }
      }
    },
    {
      "Sid": "RequireMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### Terraform Sentinel

```hcl
# sentinel.hcl - policy configuration
policy "require-encryption" {
  source            = "./policies/require-encryption.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "restrict-instance-types" {
  source            = "./policies/restrict-instance-types.sentinel"
  enforcement_level = "soft-mandatory"
}
```

## CI/CD Policy Gates

```yaml
# policy-gate.yml
- name: Evaluate Deployment Policy
  run: |
    cat > deployment-input.json << EOF
    {
      "image": "${{ env.IMAGE }}",
      "image_signed": ${{ steps.verify-sig.outputs.verified }},
      "vulnerability_scan_passed": ${{ steps.trivy.outputs.exit_code == 0 }},
      "sast_scan_passed": ${{ steps.sast.outputs.exit_code == 0 }},
      "vulnerabilities": ${{ steps.trivy.outputs.vulnerabilities }}
    }
    EOF

    opa eval \
      --data policies/ \
      --input deployment-input.json \
      "data.deployment.allow" \
      | jq -e '.result[0].expressions[0].value == true'
```
