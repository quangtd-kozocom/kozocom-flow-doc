# Environment Variables

Danh sách đầy đủ các biến môi trường.

## Required

### GitHub App

```bash
# App ID từ GitHub App settings
GITHUB_APP_ID=123456

# Private key (2 cách)
# Cách 1: Inline
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
# Cách 2: File path
GITHUB_PRIVATE_KEY_PATH=/path/to/key.pem

# Webhook secret
GITHUB_WEBHOOK_SECRET=your-secret-here
```

### Redis

```bash
# Local
REDIS_URL=redis://localhost:6379

# With password
REDIS_URL=redis://:password@localhost:6379

# Cloud (TLS)
REDIS_URL=rediss://default:password@host:6379
```

### LLM Provider

Cần ít nhất 1 trong 3:

```bash
# OpenRouter (recommended)
OPENROUTER_API_KEY=sk-or-v1-xxx
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# OpenAI
OPENAI_API_KEY=sk-xxx

# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxx
```

**Priority:** OpenRouter → OpenAI → Anthropic

## Optional

### Application

```bash
DEBUG=false
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR
APP_URL=https://your-domain.com
```

### Database

```bash
# PostgreSQL (cho config persistence)
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db

# Cache TTL (seconds)
CONFIG_CACHE_TTL=300
```

### Slack

```bash
SLACK_BOT_TOKEN=xoxb-xxx
SLACK_CHANNEL=#pr-reviews
```

### Monitoring

```bash
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
```

---

## Example .env

```bash
# ===================
# REQUIRED
# ===================
GITHUB_APP_ID=123456
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIE...\n-----END RSA PRIVATE KEY-----"
GITHUB_WEBHOOK_SECRET=abc123

OPENROUTER_API_KEY=sk-or-v1-xxx
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

REDIS_URL=redis://localhost:6379

# ===================
# OPTIONAL
# ===================
DEBUG=false
LOG_LEVEL=INFO

# SLACK_BOT_TOKEN=xoxb-xxx
# SLACK_CHANNEL=#pr-reviews

# SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
```

---

**Quay lại:** [Kiến trúc](architecture.md)
