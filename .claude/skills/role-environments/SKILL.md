---
name: role-environments
description: Multi-environment strategy and feature flags: typed per-environment config, .env layout, secrets managers, gradual rollout, kill switches, and zero-downtime expand/contract migrations. Use when setting up local/dev/staging/prod, wiring env vars or feature flags, or planning a backward-compatible schema migration.
---

# Skill: Multi-Environment and Feature Flags

## Core principle
The code is the same in every environment. What changes is the configuration.
Never `if ENV == "production"` inside business logic.

---

## Environment strategy

```
local → dev → staging → production
  │       │       │          │
  │       │       │          └── real traffic, real data
  │       │       └── mirror of prod, anonymized data
  │       └── continuous integration, test data
  └── developer machine
```

### Rules per environment

| Rule | Local | Dev | Staging | Prod |
|-------|-------|-----|---------|------|
| Real user data | ❌ | ❌ | ❌ | ✅ |
| Anonymized data | ✅ | ✅ | ✅ | ❌ |
| Automatic migrations | ✅ | ✅ | ❌ | ❌ |
| Debug logs | ✅ | ✅ | ❌ | ❌ |
| Active feature flags | All | All | Selected | Gradual |
| Manual deploy | ✅ | ❌ | ❌ | ❌ |

---

## Configuration management

### Pattern: typed config per environment

```python
# config/settings.py
from pydantic_settings import BaseSettings
from enum import Enum

class Environment(str, Enum):
    LOCAL = "local"
    DEV = "dev"
    STAGING = "staging"
    PRODUCTION = "production"

class Settings(BaseSettings):
    env: Environment = Environment.LOCAL
    debug: bool = False
    database_url: str
    redis_url: str
    api_key: str
    log_level: str = "info"

    # Values that change per environment
    @property
    def is_production(self) -> bool:
        return self.env == Environment.PRODUCTION

    @property
    def allow_migrations(self) -> bool:
        return self.env in (Environment.LOCAL, Environment.DEV)

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

### Configuration files per environment

```
.env.example          # template with no real values → committed
.env.local            # local values → in .gitignore
.env.dev              # dev values → in .gitignore or secrets manager
.env.staging          # staging values → secrets manager
.env.production       # NEVER on the filesystem → secrets manager only
```

```bash
# .env.example — committed, no real values
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
REDIS_URL=redis://localhost:6379
API_KEY=your-api-key-here
LOG_LEVEL=info
```

---

## Feature Flags

### When to use feature flags

| Case | Use |
|------|------|
| Deploy without enabling the feature | ✅ |
| A/B testing | ✅ |
| Gradual rollout (1% → 10% → 100%) | ✅ |
| Emergency kill switch | ✅ |
| Config that varies per user/segment | ✅ |
| Permanent business logic | ❌ use regular config |

### Simple implementation (no library)

```python
# domain/ports/feature_flags.py
from abc import ABC, abstractmethod

class FeatureFlags(ABC):
    @abstractmethod
    def is_enabled(self, flag: str, user_id: str | None = None) -> bool: ...

# infrastructure/adapters/env_feature_flags.py
class EnvFeatureFlags(FeatureFlags):
    def is_enabled(self, flag: str, user_id: str | None = None) -> bool:
        return os.environ.get(f"FEATURE_{flag.upper()}", "false").lower() == "true"

# usage in a use case
class CheckoutUseCase:
    def __init__(self, flags: FeatureFlags):
        self._flags = flags

    def execute(self, cart: Cart) -> Order:
        if self._flags.is_enabled("NEW_PAYMENT_FLOW", cart.user_id):
            return self._new_payment_flow(cart)
        return self._legacy_payment_flow(cart)
```

### With LaunchDarkly / GrowthBook (production)

```python
import ldclient
from ldclient.config import Config

ldclient.set_config(Config(os.environ["LAUNCHDARKLY_SDK_KEY"]))
client = ldclient.get()

def is_feature_enabled(flag: str, user_id: str) -> bool:
    context = ldclient.Context.builder(user_id).build()
    return client.variation(flag, context, default=False)
```

---

## Gradual rollout

```
Phase 1: 0% of users    → feature off, dev/staging only
Phase 2: 1% of users    → smoke test in prod with minimal real traffic
Phase 3: 10% of users   → monitor metrics, errors, latency
Phase 4: 50% of users   → A/B testing, compare metrics
Phase 5: 100% of users  → feature fully enabled
Phase 6: remove the flag → code cleanup (technical debt)
```

**Never leave old flags in the code.** Every flag has an expiration date.

---

## Migrations — strategy per environment

### Zero-downtime migrations (production)

```
❌ Renaming a column in one step:
   ALTER TABLE users RENAME COLUMN user_name TO name;
   → breaks the production code that uses user_name

✅ In three deploys:
   Deploy 1: add the new `name` column, write to both
   Deploy 2: read from `name`, keep writing to both
   Deploy 3: drop the `user_name` column
```

```python
# Expand-Contract pattern
# Step 1: Expand — add, don't change
def upgrade():
    op.add_column('users', sa.Column('name', sa.String))
    op.execute("UPDATE users SET name = user_name")

# Step 2 (next deploy): Contract — remove the old one
def upgrade():
    op.drop_column('users', 'user_name')
```

---

## Checklist before deploying to production

- [ ] Feature tested in staging with anonymized data
- [ ] Feature flag configured for gradual rollout
- [ ] Backward-compatible migrations (they don't break the previous version)
- [ ] Rollback plan defined (how do we revert if something fails?)
- [ ] Alerts configured for the feature's relevant metrics
- [ ] Runbook updated if the operation is complex

---

## Common Multi-Environment decisions

Apply the decision protocol from CLAUDE.md when facing:
- **Feature flags:** homebrew vs LaunchDarkly vs GrowthBook vs Unleash
- **Config management:** env vars vs AWS Parameter Store vs Vault
- **Staging data:** anonymization vs synthetic data vs a subset of prod
- **Deploy strategy:** blue/green vs canary vs rolling (see `role-cicd`)
