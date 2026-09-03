---
name: role-security
description: Security by design applied to code - OWASP Top 10 patterns, access control, JWT/RBAC, crypto and password hashing, input validation, secrets handling, HTTP security headers, and threat modeling. Use when writing auth, handling user input, managing secrets or env vars, reviewing a PR for vulnerabilities, or threat-modeling a feature.
---

# Skill: Security

## Base principle
Security by design. It is not a layer added at the end — it is a dimension of every architectural decision.

---

## OWASP Top 10 — applied to code

### A01 — Broken Access Control
```python
# ❌ Trusting the client
def get_order(order_id: str, user_id: str = request.query["user_id"]):
    return db.find_order(order_id)  # anyone can see any order

# ✅ Verify on the server
def get_order(order_id: str, current_user: User = Depends(get_current_user)):
    order = db.find_order(order_id)
    if order.user_id != current_user.id:
        raise ForbiddenError()
    return order
```

### A02 — Cryptographic Failures
```python
# ❌ Sensitive data in plain text or with a weak hash
password_hash = md5(password)

# ✅ Secure hash with salt
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

### A03 — Injection
```python
# ❌ SQL injection
query = f"SELECT * FROM users WHERE name = '{user_input}'"

# ✅ Parameterized
query = "SELECT * FROM users WHERE name = :name"
db.execute(query, {"name": user_input})
```

### A07 — Auth failures
- JWT tokens: validate signature + expiration + audience
- Sessions: regenerate the ID after login
- Passwords: minimum 12 chars, no absurd special-character restrictions
- MFA on accounts with elevated privileges

### A09 — Logging failures
```python
# ❌ Logging sensitive data
logger.info(f"Login: user={email} password={password}")

# ✅ Only what's needed
logger.info(f"Login attempt: user_id={user_id} success={success}")
```

---

## Secrets — non-negotiable rules

```bash
# ❌ Never in code
API_KEY = "<secret-key-here>"
DATABASE_URL = "postgres://<user>:<pass>@<host>/<db>"

# ✅ Always in env vars
import os
API_KEY = os.environ["API_KEY"]  # fails at startup if missing → intentional
```

- `.env` in `.gitignore`, always
- Secrets rotate periodically (< 90 days in production)
- Principle of least privilege: each service only holds the secrets it needs
- Use AWS Secrets Manager / Vault in production, not OS env vars

---

## Authentication and Authorization

### JWT — correct validation
```python
import jwt

def validate_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=["HS256"],       # specify the algorithm explicitly
            audience="my-api",          # validate audience
            options={"verify_exp": True} # verify expiration
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise UnauthorizedError("Expired token")
    except jwt.InvalidTokenError:
        raise UnauthorizedError("Invalid token")
```

### Basic RBAC
```python
from enum import Enum

class Permission(Enum):
    READ_USERS = "read:users"
    WRITE_USERS = "write:users"
    DELETE_USERS = "delete:users"

def require_permission(permission: Permission):
    def decorator(func):
        def wrapper(current_user: User, *args, **kwargs):
            if permission not in current_user.permissions:
                raise ForbiddenError(f"Requires permission: {permission.value}")
            return func(current_user, *args, **kwargs)
        return wrapper
    return decorator
```

---

## Input validation — at the edge

```python
# All validation at the entry point, before reaching the domain
from pydantic import BaseModel, validator, constr

class CreateUserInput(BaseModel):
    name: constr(min_length=1, max_length=100, strip_whitespace=True)
    email: str
    age: int

    @validator("email")
    def validate_email(cls, v):
        if "@" not in v or "." not in v.split("@")[-1]:
            raise ValueError("Invalid email")
        return v.lower().strip()

    @validator("age")
    def validate_age(cls, v):
        if not 0 < v < 150:
            raise ValueError("Invalid age")
        return v
```

---

## HTTP security headers

```python
# For any API/web — configure at the entry point
SECURITY_HEADERS = {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
    "Content-Security-Policy": "default-src 'self'",
    "Referrer-Policy": "strict-origin-when-cross-origin",
}
```

---

## Threat Modeling — questions to ask per feature

For every new feature that involves data or access:

1. **Who can access this?** → define authentication + authorization
2. **What if the input is malicious?** → validate + sanitize
3. **What sensitive data does it handle?** → encryption + safe logging
4. **What can go wrong in infra?** → timeouts, circuit breakers, fallbacks
5. **How do we audit access?** → access logs for sensitive data

---

## Per-PR security checklist

- [ ] No secrets in code or logs
- [ ] Input validated before use
- [ ] Authorization verified on the server
- [ ] Parameterized queries (no concatenation)
- [ ] Sensitive data encrypted at rest
- [ ] Dependencies free of known critical vulnerabilities
- [ ] Errors do not expose implementation details to the client
