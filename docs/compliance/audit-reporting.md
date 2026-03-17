# Audit & Reporting

## The Purpose of Audit Trails

Security audits serve multiple purposes:

- **Compliance** — demonstrate adherence to frameworks (SOC 2, PCI-DSS, ISO 27001)
- **Forensics** — reconstruct events during incident investigation
- **Accountability** — track who did what and when
- **Detection** — identify anomalous patterns
- **Improvement** — identify gaps and measure progress

## What to Audit

### Critical Events Requiring Audit Trails

```
Must Audit:
├── Authentication
│   ├── Login/logout (success and failure)
│   ├── MFA events
│   ├── Password changes
│   └── Session creation/termination
├── Authorization
│   ├── Access denials
│   ├── Privilege escalation
│   └── Role/permission changes
├── Data Access
│   ├── Access to sensitive data
│   ├── Bulk data exports
│   └── Data deletion
├── Configuration Changes
│   ├── System configuration changes
│   ├── Security policy changes
│   └── User account creation/deletion
├── Deployments
│   ├── Code deployments
│   ├── Infrastructure changes
│   └── Secret rotations
└── Security Events
    ├── Firewall rule changes
    ├── Certificate operations
    └── Incident response actions
```

## Audit Log Requirements

### Log Content

```json
{
  "event_id": "evt_01HN...",
  "timestamp": "2024-01-15T14:32:15.123Z",
  "event_type": "authentication",
  "action": "login_success",
  "actor": {
    "id": "user_123",
    "type": "user",
    "username": "john.doe@example.com",
    "ip_address": "203.0.113.42",
    "user_agent": "Mozilla/5.0..."
  },
  "target": {
    "type": "application",
    "id": "myapp-production"
  },
  "outcome": "success",
  "mfa_used": true,
  "session_id": "sess_abc123",
  "request_id": "req_xyz789"
}
```

### Log Integrity

Audit logs must be tamper-evident:

```bash
# AWS CloudTrail - log file validation
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/main-trail \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z

# Check if logs are valid
# Returns validation status for each log file
```

**Integrity controls:**

- Write-once, append-only log storage
- Cryptographic hash chaining (each log signs previous)
- Separate log account with restricted access
- MFA-delete protection on log buckets

## Compliance Reporting

### Automated Evidence Collection

```python
#!/usr/bin/env python3
"""Automated SOC 2 evidence collector"""

import boto3
from datetime import datetime, timedelta
import json

def collect_access_control_evidence(days=30):
    """Collect evidence for CC6.1 - Access Control"""
    iam = boto3.client('iam')
    evidence = {}

    # Users without MFA
    paginator = iam.get_paginator('list_users')
    users_without_mfa = []
    for page in paginator.paginate():
        for user in page['Users']:
            mfa = iam.list_mfa_devices(UserName=user['UserName'])
            if not mfa['MFADevices']:
                users_without_mfa.append(user['UserName'])

    evidence['users_without_mfa'] = users_without_mfa
    evidence['mfa_compliance_rate'] = (
        1 - len(users_without_mfa) / (len(users_without_mfa) + 10)
    ) * 100

    # Unused access keys
    users_with_old_keys = []
    for page in paginator.paginate():
        for user in page['Users']:
            keys = iam.list_access_keys(UserName=user['UserName'])
            for key in keys['AccessKeyMetadata']:
                if key['Status'] == 'Active':
                    last_used = iam.get_access_key_last_used(
                        AccessKeyId=key['AccessKeyId']
                    )
                    # Flag keys unused for 90+ days
                    if 'LastUsedDate' in last_used['AccessKeyLastUsed']:
                        days_since = (datetime.now(timezone.utc) -
                                     last_used['AccessKeyLastUsed']['LastUsedDate']).days
                        if days_since > 90:
                            users_with_old_keys.append({
                                'user': user['UserName'],
                                'key_id': key['AccessKeyId'],
                                'days_unused': days_since
                            })

    evidence['stale_access_keys'] = users_with_old_keys

    return evidence


def generate_compliance_report(framework="SOC2"):
    """Generate a compliance status report"""
    report = {
        "framework": framework,
        "report_date": datetime.utcnow().isoformat(),
        "period": {
            "start": (datetime.utcnow() - timedelta(days=30)).isoformat(),
            "end": datetime.utcnow().isoformat()
        },
        "controls": {}
    }

    # CC6.1 - Access Control
    report["controls"]["CC6.1"] = collect_access_control_evidence()

    return report


if __name__ == "__main__":
    report = generate_compliance_report("SOC2")
    with open(f"compliance-report-{datetime.utcnow().strftime('%Y%m')}.json", "w") as f:
        json.dump(report, f, indent=2, default=str)
    print(f"Report generated: {len(report['controls'])} controls evaluated")
```

### Security Metrics Dashboard

Track these metrics for ongoing compliance reporting:

| Metric | Formula | Target |
|--------|---------|--------|
| **Vulnerability Density** | Critical vulns / 1000 lines of code | < 0.1 |
| **Mean Time to Patch** | Avg days from CVE publish to fix | < 30 days |
| **Security Training Rate** | Staff with current security training | > 95% |
| **MFA Adoption** | Users with MFA enabled / total users | 100% |
| **Patch Compliance** | Patched systems / total systems | > 98% |
| **Incident MTTD** | Avg time to detect incidents | < 24 hours |
| **Incident MTTR** | Avg time to contain incidents | < 72 hours |
| **Scan Coverage** | Repos with security scanning / total repos | 100% |

## Security Review Calendar

| Review Type | Frequency | Owner | Evidence Collected |
|-------------|-----------|-------|-------------------|
| Access review | Quarterly | IT/Security | IAM reports, access logs |
| Vulnerability assessment | Monthly | Security | Scan reports, remediation tickets |
| Security awareness training | Annual | HR/Security | Training completion records |
| Penetration testing | Annual | Security | Pen test report |
| Incident response drill | Semi-annual | Security | Drill report, lessons learned |
| Third-party risk assessment | Annual | Procurement | Vendor assessments |
| Policy review | Annual | CISO | Updated policy documents |

## Audit Preparation Checklist

```markdown
## Audit Readiness Checklist

### Documentation
- [ ] Security policies are current and approved
- [ ] Risk register is maintained and reviewed
- [ ] Asset inventory is complete and accurate
- [ ] Data flow diagrams are up to date
- [ ] Vendor risk assessments are current

### Technical Controls
- [ ] All systems have logging enabled
- [ ] Access reviews completed (last 90 days)
- [ ] Vulnerability scans completed (last 30 days)
- [ ] Penetration test completed (last 12 months)
- [ ] MFA enabled for all privileged accounts
- [ ] Patch compliance > 95%

### Processes
- [ ] Incident response plan tested
- [ ] Security training completion > 95%
- [ ] Change management records available
- [ ] Third-party audit reports collected

### Evidence Collection
- [ ] CI/CD pipeline scan results
- [ ] Deployment records with approvals
- [ ] Security incident logs
- [ ] Access provisioning/deprovisioning records
```
