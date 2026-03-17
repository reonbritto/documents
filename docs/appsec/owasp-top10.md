# OWASP Top 10 (2021)

The OWASP Top 10 is the most recognized standard for web application security risks.

## A01: Broken Access Control

**Risk:** Users can act outside their intended permissions.

**Examples:**

- Accessing another user's account by changing the URL ID
- Privilege escalation (regular user accessing admin functions)
- CORS misconfiguration allowing unauthorized API access
- IDOR (Insecure Direct Object References)

**Prevention:**

```python
# Bad: No authorization check
@app.route('/api/users/<user_id>/profile')
def get_profile(user_id):
    return db.get_user(user_id)

# Good: Verify the requester owns the resource
@app.route('/api/users/<user_id>/profile')
@login_required
def get_profile(user_id):
    if current_user.id != user_id and not current_user.is_admin:
        abort(403)
    return db.get_user(user_id)
```

- Deny by default
- Implement access control checks server-side
- Disable directory listing
- Log access control failures and alert on repeated failures

## A02: Cryptographic Failures

**Risk:** Exposure of sensitive data due to weak or missing encryption.

**Prevention:**

```python
# Bad: Weak hashing
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()

# Good: Use bcrypt with proper cost factor
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

- Use TLS 1.2+ for all data in transit
- Encrypt sensitive data at rest (AES-256)
- Use strong hashing for passwords (bcrypt, scrypt, Argon2)
- Don't use deprecated algorithms (MD5, SHA-1, DES)
- Don't store sensitive data unnecessarily

## A03: Injection

**Risk:** Untrusted data sent to an interpreter as part of a command or query.

**Types:** SQL, NoSQL, OS command, LDAP, XPath, ORM injection.

```python
# Bad: SQL injection
query = f"SELECT * FROM users WHERE name = '{user_input}'"
cursor.execute(query)

# Good: Parameterized query
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
```

```javascript
// Bad: Command injection
const { exec } = require('child_process');
exec(`ping ${userInput}`);

// Good: Use safe APIs
const { execFile } = require('child_process');
execFile('ping', ['-c', '4', userInput]);
```

## A04: Insecure Design

**Risk:** Missing or ineffective security controls due to flawed design.

**Prevention:**

- Threat modeling during design phase (STRIDE)
- Secure design patterns and reference architectures
- Abuse case stories alongside user stories
- Security requirements in the definition of done

## A05: Security Misconfiguration

**Risk:** Insecure default configurations, incomplete setup, or unnecessary features.

**Examples:**

- Default credentials left active
- Unnecessary services or features enabled
- Detailed error messages exposed to users
- Missing security headers
- Cloud storage with public access

**Security Headers:**

```nginx
# Recommended security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

## A06: Vulnerable and Outdated Components

**Risk:** Using components with known vulnerabilities.

- Use SCA tools (Snyk, Dependabot) — see [SCA documentation](sca.md)
- Keep dependencies updated
- Monitor security advisories
- Remove unused dependencies

## A07: Identification and Authentication Failures

**Risk:** Weaknesses in authentication mechanisms.

**Prevention:**

```python
# Multi-factor authentication
# Rate limiting on login
# Account lockout after failed attempts
# Secure password requirements

from flask_limiter import Limiter

limiter = Limiter(app, default_limits=["100 per hour"])

@app.route('/login', methods=['POST'])
@limiter.limit("5 per minute")  # Rate limit login attempts
def login():
    # Verify credentials
    # Check MFA token
    # Generate secure session
    pass
```

- Implement MFA
- Don't ship with default credentials
- Implement proper password policies
- Use secure session management
- Rate limit authentication endpoints

## A08: Software and Data Integrity Failures

**Risk:** Code and infrastructure that doesn't protect against integrity violations.

- Verify software updates and dependencies (checksums, signatures)
- Use CI/CD pipeline integrity controls
- Sign artifacts (Cosign, Sigstore)
- Use lock files and verify integrity

## A09: Security Logging and Monitoring Failures

**Risk:** Insufficient logging and monitoring delays or prevents detection.

```python
import logging

security_logger = logging.getLogger('security')

# Log security-relevant events
security_logger.warning("Failed login attempt", extra={
    'user': username,
    'ip': request.remote_addr,
    'user_agent': request.user_agent.string
})

security_logger.critical("Privilege escalation attempt", extra={
    'user': current_user.id,
    'attempted_action': 'admin_access',
    'ip': request.remote_addr
})
```

- Log authentication events (success and failure)
- Log authorization failures
- Log input validation failures
- Ensure logs have enough context for forensics
- Monitor logs and alert on suspicious patterns

## A10: Server-Side Request Forgery (SSRF)

**Risk:** Application fetches remote resources without validating user-supplied URLs.

```python
# Bad: Unvalidated URL fetch
@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    response = requests.get(url)  # Can access internal services!
    return response.text

# Good: Validate and restrict
from urllib.parse import urlparse

ALLOWED_HOSTS = ['api.example.com', 'cdn.example.com']

@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    parsed = urlparse(url)

    if parsed.hostname not in ALLOWED_HOSTS:
        abort(400, "URL not allowed")
    if parsed.scheme not in ('http', 'https'):
        abort(400, "Scheme not allowed")

    response = requests.get(url, timeout=5)
    return response.text
```

- Validate and sanitize all user-supplied URLs
- Use allowlists for permitted domains
- Block access to internal/private IP ranges
- Use network segmentation to limit SSRF impact
