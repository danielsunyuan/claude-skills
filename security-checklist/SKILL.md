---
name: security-checklist
description: Security checklist, OWASP top 10 reference, and secure coding patterns. Use when reviewing code for security issues, implementing authentication, or hardening applications.
allowed-tools: Read, Grep, Glob
---

# Security Checklist

Quick reference for application security review and secure coding.

## OWASP Top 10 Quick Reference

| # | Vulnerability | Prevention |
|---|---------------|------------|
| 1 | Broken Access Control | Verify permissions on every request |
| 2 | Cryptographic Failures | Use modern algorithms, encrypt sensitive data |
| 3 | Injection | Parameterized queries, validate inputs |
| 4 | Insecure Design | Threat model, security requirements |
| 5 | Security Misconfiguration | Disable debug, remove defaults |
| 6 | Vulnerable Components | Update dependencies regularly |
| 7 | Auth Failures | Strong passwords, MFA, secure sessions |
| 8 | Data Integrity | Verify signatures, secure CI/CD |
| 9 | Logging Failures | Log security events, monitor anomalies |
| 10 | SSRF | Validate URLs, whitelist destinations |

## Security Review Checklist

### Authentication
- [ ] Passwords hashed with Argon2 or bcrypt
- [ ] Password complexity enforced
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
- [ ] Session timeout implemented
- [ ] Token rotation for long sessions

### Authorization
- [ ] RBAC/ABAC implemented
- [ ] Object-level permission checks
- [ ] Default deny policy
- [ ] Least privilege principle

### Input Validation
- [ ] All user inputs validated
- [ ] SQL injection prevented
- [ ] XSS prevented (output encoding)
- [ ] File upload restrictions
- [ ] Size limits on all inputs

### Data Protection
- [ ] Sensitive data encrypted at rest
- [ ] TLS/HTTPS enforced
- [ ] Secrets not in code
- [ ] PII minimized
- [ ] Secure deletion procedures

### Configuration
- [ ] Debug mode disabled in production
- [ ] Error messages don't leak info
- [ ] Security headers configured
- [ ] CORS properly restricted
- [ ] Default credentials changed

## Secure Coding Patterns

### SQL Injection Prevention

```python
# VULNERABLE - Never do this
query = f"SELECT * FROM users WHERE username = '{username}'"

# SAFE - Parameterized query
query = "SELECT * FROM users WHERE username = :username"
db.execute(query, {"username": username})

# SAFE - ORM
db.query(User).filter(User.username == username).first()
```

### Password Hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

### XSS Prevention

```python
from markupsafe import escape

# Always escape user input in templates
safe_output = escape(user_input)

# Jinja2 auto-escapes by default
# Use |safe filter ONLY for trusted HTML
```

### Security Headers

```python
@app.after_request
def set_security_headers(response):
    response.headers['X-Frame-Options'] = 'SAMEORIGIN'
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Content-Security-Policy'] = "default-src 'self'"
    response.headers['Strict-Transport-Security'] = 'max-age=31536000'
    return response
```

### Secure File Upload

```python
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif'}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB

def allowed_file(filename: str) -> bool:
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def handle_upload(file):
    if not allowed_file(file.filename):
        raise ValueError("File type not allowed")

    # Check actual file size
    file.seek(0, os.SEEK_END)
    if file.tell() > MAX_FILE_SIZE:
        raise ValueError("File too large")
    file.seek(0)

    # Sanitize filename
    filename = secure_filename(file.filename)
    unique_name = f"{uuid.uuid4()}_{filename}"

    return unique_name
```

### JWT Token Handling

```python
import jwt
from datetime import datetime, timedelta

SECRET_KEY = os.getenv('JWT_SECRET_KEY')  # From environment!
ALGORITHM = 'HS256'

def create_token(data: dict, expires_hours: int = 1) -> str:
    expire = datetime.utcnow() + timedelta(hours=expires_hours)
    to_encode = {**data, "exp": expire}
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise ValueError("Token expired")
    except jwt.JWTError:
        raise ValueError("Invalid token")
```

### Authorization Decorator

```python
from functools import wraps
from flask import abort, g

def require_permission(permission: str):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            if not g.user:
                abort(401)
            if not g.user.has_permission(permission):
                abort(403)
            return f(*args, **kwargs)
        return decorated
    return decorator

@app.route('/admin/users')
@require_permission('user.manage')
def manage_users():
    pass
```

## Secret Management

```python
# NEVER hardcode secrets
API_KEY = "sk_live_abc123"  # WRONG!

# Use environment variables
API_KEY = os.getenv('API_KEY')
if not API_KEY:
    raise ValueError("API_KEY not set")

# Validate required config at startup
class Config:
    SECRET_KEY = os.getenv('SECRET_KEY')
    DATABASE_URL = os.getenv('DATABASE_URL')

    @classmethod
    def validate(cls):
        required = ['SECRET_KEY', 'DATABASE_URL']
        missing = [k for k in required if not getattr(cls, k)]
        if missing:
            raise ValueError(f"Missing: {missing}")
```

## Secure Logging

```python
import re

def sanitize_for_log(data: str) -> str:
    # Mask credit cards
    data = re.sub(r'\d{4}-\d{4}-\d{4}-\d{4}', '****-****-****-****', data)
    # Mask emails
    data = re.sub(r'(\w{2})\w+@(\w+)', r'\1***@\2', data)
    return data

# Never log: passwords, tokens, PII, credit cards
```

## Rate Limiting

```python
from flask_limiter import Limiter

limiter = Limiter(app, key_func=get_remote_address)

@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    pass

@app.route('/api/data')
@limiter.limit("100 per hour")
def get_data():
    pass
```
