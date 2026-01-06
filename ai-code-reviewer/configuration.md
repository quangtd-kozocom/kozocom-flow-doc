# Configuration

AI Code Reviewer có thể được cấu hình thông qua hai cách:
1. **Environment Variables** - Cấu hình server-level, áp dụng cho tất cả repositories
2. **Repository Configuration** - File `.reviewer.yaml` trong mỗi repository, cho phép tùy chỉnh per-repo

---

## Environment Variables

### Biến bắt buộc (Required)

Các biến này **phải** được set để hệ thống hoạt động.

#### GitHub App Configuration

```bash
# GitHub App ID
# Lấy từ: GitHub App settings page
# Format: Số nguyên
GITHUB_APP_ID=123456

# GitHub App Private Key
# Lấy từ: Download khi tạo GitHub App
# Format: PEM content (với \n cho newlines) hoặc file path
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIE...\n-----END RSA PRIVATE KEY-----"
# Hoặc sử dụng file path:
# GITHUB_PRIVATE_KEY_PATH=/path/to/private-key.pem

# Webhook Secret
# Tạo khi setup GitHub App webhook
# Format: String (recommend 32+ characters)
GITHUB_WEBHOOK_SECRET=your-webhook-secret-here
```

#### Redis Configuration

```bash
# Redis URL cho Celery task queue
# Format: redis://[password@]host:port[/db]
# Local development:
REDIS_URL=redis://localhost:6379

# With password:
REDIS_URL=redis://:mypassword@localhost:6379

# Cloud (TLS enabled):
REDIS_URL=rediss://default:password@host.upstash.io:6379
```

#### LLM Provider (Ít nhất một trong các options)

```bash
# Option 1: OpenRouter (Recommended)
# Đăng ký tại: https://openrouter.ai
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxx
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# Option 2: OpenAI Direct
# Đăng ký tại: https://platform.openai.com
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxx

# Option 3: Anthropic Direct
# Đăng ký tại: https://console.anthropic.com
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxx
```

**LLM Provider Priority:**

Hệ thống sẽ sử dụng provider theo thứ tự:
1. OpenRouter (nếu `OPENROUTER_API_KEY` được set)
2. OpenAI (nếu `OPENAI_API_KEY` được set)
3. Anthropic (nếu `ANTHROPIC_API_KEY` được set)

---

### Biến tùy chọn (Optional)

#### Application Settings

```bash
# Debug mode - bật logging chi tiết
# Values: true, false
# Default: false
DEBUG=false

# Log level
# Values: DEBUG, INFO, WARNING, ERROR, CRITICAL
# Default: INFO
LOG_LEVEL=INFO

# Application URL (dùng cho OpenRouter analytics)
# Format: URL
APP_URL=https://your-domain.com
```

#### Database (Optional - cho config persistence)

```bash
# PostgreSQL connection string
# Format: postgresql+asyncpg://user:password@host:port/database
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/ai_reviewer

# Config cache TTL (seconds)
# Thời gian cache repository config trước khi fetch lại
# Default: 300 (5 minutes)
CONFIG_CACHE_TTL=300
```

#### Slack Integration

```bash
# Slack Bot Token
# Lấy từ: Slack App OAuth settings
# Format: xoxb-...
SLACK_BOT_TOKEN=xoxb-xxxx-xxxx-xxxx

# Slack Channel cho notifications
# Format: #channel-name hoặc Channel ID
# Default: #pr-reviews
SLACK_CHANNEL=#pr-reviews
```

#### Error Tracking

```bash
# Sentry DSN cho error tracking
# Lấy từ: Sentry project settings
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
```

---

### Ví dụ file .env hoàn chỉnh

```bash
# =============================================================================
# AI CODE REVIEWER - ENVIRONMENT CONFIGURATION
# =============================================================================

# -----------------------------------------------------------------------------
# APPLICATION
# -----------------------------------------------------------------------------
DEBUG=false
LOG_LEVEL=INFO
APP_URL=https://reviewer.example.com

# -----------------------------------------------------------------------------
# GITHUB APP (Required)
# -----------------------------------------------------------------------------
GITHUB_APP_ID=123456
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEA...\n-----END RSA PRIVATE KEY-----"
GITHUB_WEBHOOK_SECRET=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

# -----------------------------------------------------------------------------
# LLM PROVIDER (Required - at least one)
# -----------------------------------------------------------------------------
# OpenRouter (Recommended)
OPENROUTER_API_KEY=sk-or-v1-abcdef1234567890
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# -----------------------------------------------------------------------------
# REDIS (Required)
# -----------------------------------------------------------------------------
REDIS_URL=redis://localhost:6379

# -----------------------------------------------------------------------------
# DATABASE (Optional)
# -----------------------------------------------------------------------------
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/ai_reviewer
CONFIG_CACHE_TTL=300

# -----------------------------------------------------------------------------
# SLACK (Optional)
# -----------------------------------------------------------------------------
SLACK_BOT_TOKEN=xoxb-xxxx-xxxx-xxxx
SLACK_CHANNEL=#code-reviews

# -----------------------------------------------------------------------------
# MONITORING (Optional)
# -----------------------------------------------------------------------------
SENTRY_DSN=https://abc123@o123456.ingest.sentry.io/1234567
```

---

## Repository Configuration (.reviewer.yaml)

Mỗi repository có thể có file `.reviewer.yaml` ở root để tùy chỉnh behavior của AI Code Reviewer cho repository đó.

### Vị trí file

```
your-repository/
├── .reviewer.yaml    ← File configuration
├── src/
├── tests/
└── ...
```

### Schema đầy đủ

```yaml
# =============================================================================
# AI CODE REVIEWER - REPOSITORY CONFIGURATION
# =============================================================================

# -----------------------------------------------------------------------------
# LANGUAGE SETTINGS
# -----------------------------------------------------------------------------
# Ngôn ngữ cho review comments
# Values: en (English), vi (Vietnamese), ja (Japanese)
# Default: en
language: "en"

# -----------------------------------------------------------------------------
# REVIEW SETTINGS
# -----------------------------------------------------------------------------
reviews:
  # ---------------------------------------------------------------------------
  # Review Profile
  # ---------------------------------------------------------------------------
  # Độ nghiêm ngặt của review
  # Values: chill, default, strict
  # Default: default
  profile: "default"
  
  # ---------------------------------------------------------------------------
  # Agent Selection
  # ---------------------------------------------------------------------------
  # Chọn agents sẽ chạy
  # Available: security, logic, style
  # Default: all agents
  agents:
    - security
    - logic
    - style
  
  # ---------------------------------------------------------------------------
  # Filtering Settings
  # ---------------------------------------------------------------------------
  # Ngưỡng confidence tối thiểu để post comment
  # Range: 0.0 - 1.0
  # Default: 0.7
  confidence_threshold: 0.7
  
  # Số lượng comments tối đa per file
  # Default: 10
  max_comments_per_file: 10
  
  # ---------------------------------------------------------------------------
  # Auto-review Settings
  # ---------------------------------------------------------------------------
  auto_review:
    # Bật/tắt auto-review
    # Default: true
    enabled: true
    
    # Review draft PRs
    # Default: false
    drafts: false
    
    # Skip PRs có title chứa các keywords này
    # Default: ["[WIP]", "[SKIP REVIEW]"]
    skip_keywords:
      - "[WIP]"
      - "[SKIP REVIEW]"
      - "[NO REVIEW]"
      - "[DRAFT]"
    
    # Chỉ review PRs merge vào các branches này
    # Default: [] (all branches)
    base_branches:
      - main
      - master
      - develop
    
    # Bỏ qua PRs từ các authors này
    # Default: ["dependabot[bot]", "renovate[bot]"]
    ignore_authors:
      - dependabot[bot]
      - renovate[bot]
      - github-actions[bot]
      - my-deploy-bot
  
  # ---------------------------------------------------------------------------
  # Path-specific Instructions
  # ---------------------------------------------------------------------------
  # Custom instructions cho các paths cụ thể
  path_instructions:
    - path: "src/api/**/*.py"
      instructions: |
        Focus on:
        - Input validation and sanitization
        - Authentication and authorization
        - Rate limiting
        - Error response format
    
    - path: "src/models/**/*.py"
      instructions: |
        Focus on:
        - Database field validation
        - Index optimization
        - N+1 query prevention
        - Migration safety
    
    - path: "src/services/**/*.py"
      instructions: |
        Focus on:
        - Business logic correctness
        - Transaction handling
        - External service error handling
        - Idempotency
    
    - path: "tests/**/*.py"
      instructions: |
        Focus on:
        - Test coverage completeness
        - Meaningful assertions
        - Edge case coverage
        - Mock usage correctness

# -----------------------------------------------------------------------------
# IGNORE PATTERNS
# -----------------------------------------------------------------------------
# Glob patterns cho files/folders sẽ không được review
ignore:
  # Lock files
  - "**/package-lock.json"
  - "**/yarn.lock"
  - "**/pnpm-lock.yaml"
  - "**/Pipfile.lock"
  - "**/poetry.lock"
  - "**/composer.lock"
  - "**/Gemfile.lock"
  - "**/Cargo.lock"
  
  # Generated files
  - "**/migrations/**"
  - "**/*.generated.*"
  - "**/*.g.dart"
  - "**/*.freezed.dart"
  - "**/*.pb.go"
  - "**/*.pb.ts"
  
  # Minified files
  - "**/*.min.js"
  - "**/*.min.css"
  - "**/*.bundle.js"
  
  # Build outputs
  - "**/dist/**"
  - "**/build/**"
  - "**/out/**"
  - "**/.next/**"
  - "**/.nuxt/**"
  
  # Dependencies
  - "**/node_modules/**"
  - "**/vendor/**"
  - "**/.venv/**"
  - "**/venv/**"
  
  # IDE/Editor
  - "**/.idea/**"
  - "**/.vscode/**"
  - "**/*.swp"
  - "**/*.swo"
  
  # Test fixtures/snapshots
  - "**/__snapshots__/**"
  - "**/fixtures/**"
  - "**/*.snap"
  
  # Assets
  - "**/*.png"
  - "**/*.jpg"
  - "**/*.jpeg"
  - "**/*.gif"
  - "**/*.svg"
  - "**/*.ico"
  - "**/*.woff"
  - "**/*.woff2"
  - "**/*.ttf"
  - "**/*.eot"
  
  # Documentation (optional - remove if you want docs reviewed)
  - "**/docs/**"
  - "**/*.md"

# -----------------------------------------------------------------------------
# CHAT SETTINGS
# -----------------------------------------------------------------------------
chat:
  # Bật/tắt chat commands
  # Default: true
  enabled: true
  
  # Giới hạn commands được phép
  # Available: help, fix, explain, tests
  # Default: all commands
  allowed_commands:
    - help
    - fix
    - explain
    - tests
```

---

## Chi tiết các Configuration Options

### Language (`language`)

Cấu hình ngôn ngữ cho review comments.

| Value | Ngôn ngữ | Ví dụ output |
|-------|----------|--------------|
| `en` | English | "Potential SQL injection vulnerability detected" |
| `vi` | Tiếng Việt | "Phát hiện lỗ hổng SQL injection tiềm ẩn" |
| `ja` | 日本語 | "SQLインジェクションの脆弱性が検出されました" |

**Ví dụ:**
```yaml
language: "vi"
```

---

### Review Profile (`reviews.profile`)

Profile quyết định độ nghiêm ngặt của review.

#### `chill` Profile

Dành cho prototypes, hackathons, hoặc khi muốn giảm noise.

| Setting | Value |
|---------|-------|
| Severities reported | `critical`, `warning` only |
| Confidence threshold | 0.85 |
| Max comments per file | 5 |
| Style checking | Disabled |

#### `default` Profile

Cân bằng giữa thoroughness và noise. Phù hợp cho hầu hết projects.

| Setting | Value |
|---------|-------|
| Severities reported | All |
| Confidence threshold | 0.7 |
| Max comments per file | 10 |
| Style checking | Enabled |

#### `strict` Profile

Dành cho critical systems, security-sensitive code.

| Setting | Value |
|---------|-------|
| Severities reported | All |
| Confidence threshold | 0.6 |
| Max comments per file | 20 |
| Style checking | Strict |
| Additional checks | Enabled |

**Ví dụ:**
```yaml
reviews:
  profile: "strict"
```

---

### Agent Selection (`reviews.agents`)

Chọn agents sẽ chạy trong review.

| Agent | Focus | Khi nào disable |
|-------|-------|-----------------|
| `security` | SQL injection, XSS, secrets, etc. | Không nên disable |
| `logic` | Null refs, edge cases, errors | Không nên disable |
| `style` | Naming, formatting, dead code | Khi có linter riêng |

**Ví dụ - Chỉ chạy security và logic:**
```yaml
reviews:
  agents:
    - security
    - logic
```

**Ví dụ - Chỉ chạy security:**
```yaml
reviews:
  agents:
    - security
```

---

### Confidence Threshold (`reviews.confidence_threshold`)

Ngưỡng confidence tối thiểu để post comment.

| Threshold | Behavior |
|-----------|----------|
| 0.9 | Rất ít comments, chỉ những vấn đề rõ ràng |
| 0.8 | Ít comments, độ chính xác cao |
| 0.7 | Cân bằng (default) |
| 0.6 | Nhiều comments hơn, có thể có false positives |
| 0.5 | Rất nhiều comments, nhiều noise |

**Ví dụ:**
```yaml
reviews:
  confidence_threshold: 0.8  # Chỉ post comments có confidence >= 80%
```

---

### Auto-review Settings (`reviews.auto_review`)

#### `enabled`

Bật/tắt automatic review khi PR được tạo/cập nhật.

```yaml
reviews:
  auto_review:
    enabled: false  # Tắt auto-review, chỉ review khi được yêu cầu
```

#### `drafts`

Có review draft PRs hay không.

```yaml
reviews:
  auto_review:
    drafts: true  # Review cả draft PRs
```

#### `skip_keywords`

Danh sách keywords trong PR title sẽ skip review.

```yaml
reviews:
  auto_review:
    skip_keywords:
      - "[WIP]"
      - "[SKIP REVIEW]"
      - "[HOTFIX]"      # Custom: skip hotfixes
      - "[REVERT]"      # Custom: skip reverts
```

**Matching behavior:**
- Case-insensitive
- Substring match (không cần exact match)
- Ví dụ: `[WIP]` sẽ match `[WIP] Add feature` và `Feature [WIP]`

#### `base_branches`

Chỉ review PRs merge vào các branches được chỉ định.

```yaml
reviews:
  auto_review:
    base_branches:
      - main
      - develop
      - release/*  # Wildcard supported
```

**Use cases:**
- Chỉ review PRs vào production branches
- Bỏ qua PRs giữa feature branches

#### `ignore_authors`

Bỏ qua PRs từ các authors cụ thể.

```yaml
reviews:
  auto_review:
    ignore_authors:
      - dependabot[bot]
      - renovate[bot]
      - github-actions[bot]
      - my-ci-bot
```

**Use cases:**
- Bỏ qua dependency update PRs từ bots
- Bỏ qua automated PRs từ CI/CD

---

### Path Instructions (`reviews.path_instructions`)

Cung cấp custom instructions cho AI khi review các paths cụ thể.

**Format:**
```yaml
reviews:
  path_instructions:
    - path: "glob/pattern/**/*.ext"
      instructions: |
        Multi-line instructions
        for the AI to follow
```

**Glob pattern syntax:**

| Pattern | Matches |
|---------|---------|
| `*` | Any characters except `/` |
| `**` | Any characters including `/` (recursive) |
| `?` | Single character |
| `[abc]` | Character class |
| `{a,b}` | Alternatives |

**Ví dụ chi tiết:**

```yaml
reviews:
  path_instructions:
    # API layer - focus on security
    - path: "src/api/**/*.py"
      instructions: |
        This is the API layer. Focus on:
        1. Input validation - all user inputs must be validated
        2. Authentication - ensure proper auth checks
        3. Authorization - verify permission checks
        4. Rate limiting - check for rate limit implementation
        5. Error handling - don't expose internal errors to clients
        
    # Database layer - focus on performance
    - path: "src/repositories/**/*.py"
      instructions: |
        This is the database layer. Focus on:
        1. N+1 queries - look for loops with DB calls
        2. Missing indexes - suggest indexes for frequent queries
        3. Transaction handling - ensure proper transaction boundaries
        4. Connection management - check for connection leaks
        
    # Frontend components - focus on UX
    - path: "src/components/**/*.tsx"
      instructions: |
        This is a React component. Focus on:
        1. Accessibility - ARIA labels, keyboard navigation
        2. Performance - unnecessary re-renders, memo usage
        3. Error boundaries - proper error handling
        4. Loading states - user feedback during async operations
        
    # Configuration files - be extra careful
    - path: "config/**/*"
      instructions: |
        This is a configuration file. Be extra careful about:
        1. Secrets - no hardcoded credentials
        2. Environment-specific values - should use env vars
        3. Security settings - verify secure defaults
```

---

### Ignore Patterns (`ignore`)

Glob patterns cho files/folders sẽ không được review.

**Common patterns:**

```yaml
ignore:
  # By file type
  - "**/*.min.js"           # Minified JS
  - "**/*.generated.ts"     # Generated TypeScript
  - "**/*.pb.go"            # Protocol buffer generated
  
  # By directory
  - "**/node_modules/**"    # NPM dependencies
  - "**/vendor/**"          # Vendored dependencies
  - "**/dist/**"            # Build output
  - "**/migrations/**"      # Database migrations
  
  # By filename
  - "**/package-lock.json"  # Lock files
  - "**/.env*"              # Environment files
  
  # Specific paths
  - "legacy/**"             # Legacy code
  - "scripts/deploy.sh"     # Specific file
```

**Tips:**
- Sử dụng `**` để match recursive
- Patterns are case-sensitive
- Có thể combine multiple patterns

---

### Chat Settings (`chat`)

#### `enabled`

Bật/tắt chat commands.

```yaml
chat:
  enabled: false  # Disable all chat commands
```

#### `allowed_commands`

Giới hạn commands được phép sử dụng.

```yaml
chat:
  enabled: true
  allowed_commands:
    - help
    - explain
    # - fix      # Disabled
    # - tests    # Disabled
```

**Available commands:**

| Command | Description |
|---------|-------------|
| `help` | Hiển thị help message |
| `fix` | Generate code fix suggestion |
| `explain` | Giải thích chi tiết về issue |
| `tests` | Generate unit tests |

---

## Configuration Precedence

Khi có conflict giữa các sources, thứ tự ưu tiên như sau:

1. **Repository config** (`.reviewer.yaml`) - Highest priority
2. **Environment variables** - Default values
3. **Built-in defaults** - Lowest priority

**Ví dụ:**
- Environment: `LOG_LEVEL=INFO`
- `.reviewer.yaml`: `language: "vi"`
- Result: Log level = INFO, Language = Vietnamese

---

## Validation và Errors

### Config Validation

Khi load `.reviewer.yaml`, hệ thống validate:

1. **YAML syntax** - File phải là valid YAML
2. **Schema validation** - Các fields phải đúng type
3. **Value validation** - Values phải trong allowed range

### Error Handling

Nếu config invalid:

1. **Syntax error**: Sử dụng default config, log warning
2. **Invalid value**: Sử dụng default cho field đó, log warning
3. **Missing file**: Sử dụng default config (không phải error)

**Ví dụ log:**
```
WARNING: Invalid confidence_threshold value '1.5' in .reviewer.yaml, using default 0.7
WARNING: Unknown agent 'performance' in .reviewer.yaml, ignoring
```

---

## Best Practices

### 1. Start với Default, Tune dần

```yaml
# Bắt đầu đơn giản
language: "en"
reviews:
  profile: "default"
```

Sau đó tune dựa trên feedback từ team.

### 2. Sử dụng Path Instructions cho Critical Paths

```yaml
reviews:
  path_instructions:
    - path: "src/auth/**/*"
      instructions: "This is authentication code. Be extra strict about security."
```

### 3. Ignore Generated Files

```yaml
ignore:
  - "**/*.generated.*"
  - "**/migrations/**"
  - "**/*.pb.go"
```

### 4. Customize cho Team Workflow

```yaml
reviews:
  auto_review:
    skip_keywords:
      - "[WIP]"
      - "[PAIR]"  # Skip khi pair programming
    ignore_authors:
      - dependabot[bot]
      - our-release-bot
```

### 5. Version Control .reviewer.yaml

Commit `.reviewer.yaml` vào repository để:
- Team members có cùng config
- Track changes qua git history
- Review config changes trong PRs
