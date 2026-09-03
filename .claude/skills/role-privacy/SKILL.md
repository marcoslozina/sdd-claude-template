---
name: role-privacy
description: Security and privacy in code: sensitive-data classification, keeping secrets out of source, PII-safe logging and API responses, field-level encryption, data minimization and retention, secret scanning (pre-commit, gitleaks), and leaked-secret response. Use when handling PII, credentials, or user data, or reviewing code for leaks.
---

# Skill: Security and Privacy in Code

## Core principle
Privacy by design. Privacy isn't added at the end — it's a design constraint from the start.
A data leak isn't just a bug. It's an incident with legal and trust consequences.

---

## Classification of sensitive data

Before writing any code that handles data, classify it:

| Level | Data type | Examples | Handling |
|-------|-------------|---------|-------------|
| 🔴 Critical | Credentials | Passwords, API keys, tokens, certificates | Never in code, logs, or the DB without hashing |
| 🔴 Critical | Sensitive PII | National ID, passport, medical data, biometrics | Encrypted at rest, audited access |
| 🟡 Sensitive | Basic PII | Name, email, phone, address | Encrypted in transit, never log it in full |
| 🟡 Sensitive | Financial | Cards, bank account numbers, transaction amounts | PCI-DSS if applicable, tokenize |
| 🟢 Internal | Business data | Internal IDs, metrics | Role-based access control |
| ⚪ Public | Public data | Prices, catalogs | No special restrictions |

---

## Code rules — non-negotiable

### Secrets — automatic detection

```python
# ❌ Patterns that must NEVER appear in code
API_KEY = "sk-..."
SECRET = "eyJ..."
PASSWORD = "hunter2"
DATABASE_URL = "postgres://user:pass@..."

# ✅ Always from the environment
import os
API_KEY = os.environ["API_KEY"]          # fails at startup if missing → intentional
DATABASE_URL = os.environ["DATABASE_URL"]
```

**Patterns to detect in code review (regex):**
```
(api[_-]?key|secret|password|token|pwd)\s*=\s*["'][^"']{8,}["']
(sk-|pk_live_|Bearer\s+ey)[A-Za-z0-9+/]{20,}
postgres://[^:]+:[^@]+@
```

### PII — never log it in full

```python
# ❌ Full PII in logs
logger.info(f"User logged in: {user.email} from {request.ip}")
logger.error(f"Payment failed for card {card_number}")

# ✅ Only what's needed for traceability
logger.info(f"User logged in: user_id={user.id}")
logger.error(f"Payment failed: user_id={user.id} last4={card_number[-4:]}")
```

### Masking data in responses

```python
# ❌ Exposing unnecessary data in the API
return UserResponse(
    id=user.id,
    email=user.email,
    password_hash=user.password_hash,  # NEVER
    internal_score=user.risk_score,    # internal data
)

# ✅ Only what the client needs
return UserResponse(
    id=user.id,
    email=user.email,
    name=user.name,
)
```

### Encrypting sensitive data in the DB

```python
from cryptography.fernet import Fernet

class EncryptedField:
    def __init__(self, key: bytes):
        self._fernet = Fernet(key)

    def encrypt(self, value: str) -> str:
        return self._fernet.encrypt(value.encode()).decode()

    def decrypt(self, value: str) -> str:
        return self._fernet.decrypt(value.encode()).decode()

# National ID, card number, medical data → always encrypted in the DB
```

---

## Detecting secrets before commit

### Pre-commit hook (add it to the project)

```bash
# .git/hooks/pre-commit or via the pre-commit framework
#!/bin/bash
echo "🔍 Scanning for secrets..."

# Patterns that block the commit
patterns=(
  'api[_-]?key\s*=\s*["\x27][^"\x27]{8,}'
  'secret\s*=\s*["\x27][^"\x27]{8,}'
  'password\s*=\s*["\x27][^"\x27]{4,}'
  'sk-[A-Za-z0-9]{20,}'
  'pk_live_[A-Za-z0-9]+'
  'postgres://[^:]+:[^@]+@'
  'mysql://[^:]+:[^@]+@'
  'BEGIN (RSA |EC )?PRIVATE KEY'
)

for pattern in "${patterns[@]}"; do
  if git diff --cached | grep -qiE "$pattern"; then
    echo "❌ SECRET DETECTED: pattern '$pattern'"
    echo "Remove the secret and use environment variables."
    exit 1
  fi
done

echo "✅ No secrets detected"
```

### With gitleaks (recommended for CI)

```yaml
# .github/workflows/ci.yml — add a job
secret-scan:
  name: Secret Scan
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    - uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Privacy in APIs

### Data minimization — ask only for what you need

```python
# ❌ Receive and store everything
class RegistrationInput(BaseModel):
    name: str
    email: str
    phone: str
    birth_date: date
    address: str
    # ... 20 more fields we don't use

# ✅ Only what the use case needs
class RegistrationInput(BaseModel):
    name: str
    email: str
```

### Data retention — an explicit policy

```python
# Data that must be deleted or anonymized after N days
class DataRetentionPolicy:
    AUDIT_LOGS_DAYS = 90
    SESSION_TOKENS_DAYS = 30
    DELETED_USER_PII_DAYS = 7    # after deletion, anonymize the PII
    ANALYTICS_RAW_DAYS = 365
```

### Anonymization for logs and analytics

```python
import hashlib

def anonymize_email(email: str) -> str:
    # Identifiable internally but not reversible externally
    return hashlib.sha256(email.encode()).hexdigest()[:12]

def mask_ip(ip: str) -> str:
    # Keep only the first 3 octets
    parts = ip.split(".")
    return f"{'.'.join(parts[:3])}.0"
```

---

## Per-feature privacy checklist

Before implementing any feature that handles user data:

- [ ] What data do I actually need? Can I minimize it?
- [ ] Is the sensitive data encrypted at rest?
- [ ] Does the data always travel over HTTPS?
- [ ] Are the logs free of full PII?
- [ ] Do the API responses avoid exposing unnecessary fields?
- [ ] Is there a defined retention policy?
- [ ] Can users delete their data? (right to erasure)
- [ ] Is there an audit trail for access to sensitive data?
- [ ] Are the secrets in env vars / a secrets manager?
- [ ] Does the new code pass the secret scan?

---

## What to do if a committed secret is detected

```bash
# 1. REVOKE the secret IMMEDIATELY (before anything else)
#    → Rotate the API key, change the password, invalidate the token

# 2. Remove it from the git history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/file" \
  --prune-empty --tag-name-filter cat -- --all

# Or with BFG (faster)
bfg --delete-files file-with-secret.env
git push --force

# 3. Notify the team
# 4. Audit whether the secret was used by third parties
```

**Never assume a committed secret went unseen**, even if the repo is private.
