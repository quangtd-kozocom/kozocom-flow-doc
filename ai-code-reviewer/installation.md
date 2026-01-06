# Installation

Hướng dẫn chi tiết cài đặt và triển khai AI Code Reviewer từ đầu đến cuối.

## Yêu cầu hệ thống

### Software Requirements

| Component | Version | Ghi chú |
|-----------|---------|---------|
| Python | 3.13+ | Khuyến nghị sử dụng pyenv để quản lý versions |
| Redis | 6.0+ | Dùng cho task queue (Celery) |
| Git | 2.0+ | Để clone repository |

### Package Manager

Khuyến nghị sử dụng [uv](https://github.com/astral-sh/uv) - package manager nhanh và hiện đại cho Python:

```bash
# Install uv (macOS/Linux)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install uv (Windows)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Verify installation
uv --version
```

Nếu không muốn dùng uv, có thể sử dụng pip truyền thống.

### External Services

| Service | Bắt buộc | Mô tả |
|---------|----------|-------|
| GitHub App | Yes | Để nhận webhooks và post comments |
| LLM Provider | Yes | OpenRouter, OpenAI, hoặc Anthropic |
| Redis | Yes | Task queue cho background processing |
| PostgreSQL | No | Lưu trữ config (optional) |
| Slack | No | Notifications (optional) |
| Sentry | No | Error tracking (optional) |

---

## Bước 1: Clone Repository

```bash
# Clone repository
git clone https://github.com/your-org/ai-code-review-kozocom.git
cd ai-code-review-kozocom

# Kiểm tra structure
ls -la
```

**Expected structure:**
```
ai-code-review-kozocom/
├── src/
│   ├── app/           # FastAPI application
│   ├── agents/        # LangGraph AI agents
│   ├── chat/          # Chat command handlers
│   ├── core/          # Core utilities
│   └── workers/       # Celery tasks
├── scripts/           # Startup scripts
├── tests/             # Test files
├── .env.example       # Environment template
├── pyproject.toml     # Project dependencies
├── Makefile           # Development commands
└── README.md
```

---

## Bước 2: Cài đặt Dependencies

### Sử dụng uv (Recommended)

```bash
# Tạo virtual environment và install dependencies
uv sync

# Activate virtual environment
source .venv/bin/activate  # Linux/macOS
# hoặc
.venv\Scripts\activate     # Windows
```

### Sử dụng pip

```bash
# Tạo virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -e .
```

### Verify Installation

```bash
# Kiểm tra các packages đã được install
python -c "import fastapi; import celery; import langchain; print('All packages installed successfully!')"
```

---

## Bước 3: Cài đặt Redis

Redis được sử dụng làm message broker cho Celery task queue.

### Option 1: Local Redis (Development)

**macOS (Homebrew):**
```bash
brew install redis
brew services start redis

# Verify
redis-cli ping
# Expected: PONG
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Verify
redis-cli ping
```

**Docker:**
```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine

# Verify
docker exec redis redis-cli ping
```

### Option 2: Cloud Redis (Production)

**Upstash (Recommended - Free tier available):**

1. Đăng ký tại [upstash.com](https://upstash.com)
2. Tạo Redis database
3. Copy connection string (format: `rediss://...`)

**AWS ElastiCache:**
- Tạo Redis cluster trong AWS Console
- Đảm bảo security group cho phép access từ application

**Redis URL Format:**
```bash
# Local
REDIS_URL=redis://localhost:6379

# With password
REDIS_URL=redis://:password@localhost:6379

# Cloud (TLS)
REDIS_URL=rediss://default:password@host:port
```

---

## Bước 4: Tạo GitHub App

GitHub App là cách để AI Code Reviewer tương tác với GitHub repositories.

### 4.1 Tạo App

1. Truy cập [GitHub Developer Settings](https://github.com/settings/apps)
2. Click **"New GitHub App"**
3. Điền thông tin:

| Field | Value | Ghi chú |
|-------|-------|---------|
| GitHub App name | `AI Code Reviewer` | Tên unique trên GitHub |
| Homepage URL | `https://your-domain.com` | URL của app |
| Webhook URL | `https://your-domain.com/api/v1/webhooks/github` | Endpoint nhận events |
| Webhook secret | `<generate-random-string>` | Dùng để verify webhooks |

**Generate webhook secret:**
```bash
# Linux/macOS
openssl rand -hex 32

# Python
python -c "import secrets; print(secrets.token_hex(32))"
```

### 4.2 Cấu hình Permissions

Trong phần **"Permissions"**, set các quyền sau:

**Repository permissions:**

| Permission | Access | Mục đích |
|------------|--------|----------|
| Contents | Read | Đọc file contents và diffs |
| Metadata | Read | Đọc repository metadata |
| Pull requests | Read & Write | Đọc PR info, post comments |
| Issues | Read & Write | Đọc/write issue comments (cho chat) |

**Organization permissions:**
- Không cần

**Account permissions:**
- Không cần

### 4.3 Subscribe to Events

Trong phần **"Subscribe to events"**, check các events:

| Event | Mục đích |
|-------|----------|
| Pull request | Trigger review khi PR opened/updated |
| Issue comment | Nhận chat commands từ PR comments |
| Pull request review comment | Nhận replies vào review comments |

### 4.4 Generate Private Key

1. Sau khi tạo app, scroll xuống phần **"Private keys"**
2. Click **"Generate a private key"**
3. File `.pem` sẽ được download tự động
4. Lưu file này an toàn - bạn sẽ cần nó cho configuration

### 4.5 Install App on Repositories

1. Trong app settings, click **"Install App"** ở sidebar
2. Chọn account/organization
3. Chọn repositories muốn enable:
   - **All repositories**: Tất cả repos (bao gồm future repos)
   - **Only select repositories**: Chọn specific repos
4. Click **"Install"**

### 4.6 Lấy App ID và Installation ID

**App ID:**
- Hiển thị ở đầu trang app settings
- Ví dụ: `123456`

**Installation ID:**
- Sau khi install, vào **"Install App"** → click vào installation
- URL sẽ có format: `https://github.com/settings/installations/12345678`
- Installation ID là số cuối: `12345678`

---

## Bước 5: Cấu hình LLM Provider

AI Code Reviewer cần một LLM provider để chạy các AI agents.

### Option 1: OpenRouter (Recommended)

OpenRouter cung cấp access đến 200+ models qua một API duy nhất.

1. Đăng ký tại [openrouter.ai](https://openrouter.ai)
2. Vào **"Keys"** → **"Create Key"**
3. Copy API key (format: `sk-or-v1-...`)

**Recommended models:**

| Model | Cost | Speed | Quality |
|-------|------|-------|---------|
| `anthropic/claude-3.5-sonnet` | $$ | Fast | Excellent |
| `anthropic/claude-3-opus` | $$$ | Slow | Best |
| `openai/gpt-4-turbo` | $$ | Fast | Excellent |
| `openai/gpt-4o` | $$ | Fast | Excellent |
| `google/gemini-pro-1.5` | $ | Fast | Good |

### Option 2: OpenAI Direct

1. Đăng ký tại [platform.openai.com](https://platform.openai.com)
2. Vào **"API Keys"** → **"Create new secret key"**
3. Copy API key (format: `sk-...`)

### Option 3: Anthropic Direct

1. Đăng ký tại [console.anthropic.com](https://console.anthropic.com)
2. Vào **"API Keys"** → **"Create Key"**
3. Copy API key (format: `sk-ant-...`)

### Provider Priority

Hệ thống sẽ sử dụng provider theo thứ tự ưu tiên:
1. OpenRouter (nếu `OPENROUTER_API_KEY` được set)
2. OpenAI (nếu `OPENAI_API_KEY` được set)
3. Anthropic (nếu `ANTHROPIC_API_KEY` được set)

---

## Bước 6: Cấu hình Environment

### 6.1 Tạo file .env

```bash
cp .env.example .env
```

### 6.2 Điền các giá trị

Mở file `.env` và điền các giá trị:

```bash
# =============================================================================
# APPLICATION
# =============================================================================
DEBUG=false
LOG_LEVEL=INFO

# =============================================================================
# GITHUB APP (Required)
# =============================================================================
# App ID từ GitHub App settings
GITHUB_APP_ID=123456

# Private key - có 2 cách:

# Cách 1: Inline (escape newlines)
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIE...\n...\n-----END RSA PRIVATE KEY-----"

# Cách 2: File path
# GITHUB_PRIVATE_KEY_PATH=/path/to/private-key.pem

# Webhook secret đã tạo ở bước 4.1
GITHUB_WEBHOOK_SECRET=your-webhook-secret-here

# =============================================================================
# LLM PROVIDER (At least one required)
# =============================================================================
# OpenRouter (Recommended)
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxx
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# OpenAI (Alternative)
# OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxx

# Anthropic (Alternative)
# ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxx

# =============================================================================
# REDIS (Required)
# =============================================================================
# Local development
REDIS_URL=redis://localhost:6379

# Cloud (Upstash example)
# REDIS_URL=rediss://default:password@xxx.upstash.io:6379

# =============================================================================
# OPTIONAL SERVICES
# =============================================================================
# PostgreSQL - for config persistence
# DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/ai_reviewer

# Slack notifications
# SLACK_BOT_TOKEN=xoxb-...
# SLACK_CHANNEL=#pr-reviews

# Sentry error tracking
# SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
```

### 6.3 Xử lý Private Key

**Option A: Inline trong .env**

Convert private key thành single line:
```bash
# Linux/macOS
cat private-key.pem | awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}'
```

Copy output và paste vào `GITHUB_PRIVATE_KEY`.

**Option B: File path**

```bash
# Đặt file ở location an toàn
cp ~/Downloads/your-app.private-key.pem /etc/ai-reviewer/private-key.pem
chmod 600 /etc/ai-reviewer/private-key.pem

# Trong .env
GITHUB_PRIVATE_KEY_PATH=/etc/ai-reviewer/private-key.pem
```

### 6.4 Verify Configuration

```bash
# Kiểm tra env được load đúng
python -c "
from src.app.config import settings
print(f'GitHub App ID: {settings.GITHUB_APP_ID}')
print(f'Redis URL: {settings.REDIS_URL}')
print(f'LLM configured: {bool(settings.OPENROUTER_API_KEY or settings.OPENAI_API_KEY)}')
"
```

---

## Bước 7: Khởi động Services

### Development Mode

**Terminal 1 - API Server:**
```bash
make dev
# hoặc
uvicorn src.app.main:app --reload --host 0.0.0.0 --port 8000
```

**Terminal 2 - Celery Worker:**
```bash
make worker
# hoặc
celery -A src.workers.celery_app worker --loglevel=info
```

### Sử dụng Start Script (Recommended)

```bash
# Start cả API và Worker
./scripts/start.sh

# Start với ngrok (expose local server)
./scripts/start.sh --ngrok

# Start trong tmux (split panes)
./scripts/start.sh --tmux

# Development mode (API only, hot-reload)
./scripts/start.sh --dev
```

### Verify Services Running

```bash
# Check API
curl http://localhost:8000/health
# Expected: {"status": "healthy"}

# Check API docs
open http://localhost:8000/docs
```

---

## Bước 8: Expose Webhook Endpoint

GitHub cần gửi webhooks đến một public URL. Trong development, bạn cần expose local server.

### Option 1: ngrok (Recommended for Development)

```bash
# Install ngrok
# macOS
brew install ngrok

# Authenticate (one-time)
ngrok config add-authtoken YOUR_AUTH_TOKEN

# Start tunnel
ngrok http 8000
```

Copy URL từ ngrok (ví dụ: `https://abc123.ngrok.io`) và update GitHub App webhook URL:
```
https://abc123.ngrok.io/api/v1/webhooks/github
```

### Option 2: Cloudflare Tunnel

```bash
# Install cloudflared
brew install cloudflare/cloudflare/cloudflared

# Start tunnel
cloudflared tunnel --url http://localhost:8000
```

### Option 3: Production Deployment

Deploy lên cloud provider với public URL:
- **Railway**: `your-app.up.railway.app`
- **Render**: `your-app.onrender.com`
- **AWS/GCP/Azure**: Custom domain với load balancer

---

## Bước 9: Test Installation

### 9.1 Test Webhook Delivery

1. Vào GitHub App settings → **"Advanced"**
2. Scroll xuống **"Recent Deliveries"**
3. Click **"Redeliver"** trên một delivery gần đây
4. Kiểm tra response status là `200`

### 9.2 Test với Real PR

1. Tạo một branch mi trong repository đã install app:
   ```bash
   git checkout -b test/ai-reviewer
   ```

2. Tạo một file với intentional issues:
   ```python
   # test_file.py
   import os
   
   def get_user(user_id):
       query = f"SELECT * FROM users WHERE id = {user_id}"  # SQL Injection!
       password = "hardcoded_password_123"  # Hardcoded secret!
       return query
   ```

3. Commit và push:
   ```bash
   git add .
   git commit -m "Test AI Code Reviewer"
   git push -u origin test/ai-reviewer
   ```

4. Tạo Pull Request trên GitHub

5. Đợi vài phút và kiểm tra:
   - Comment "Review started..." xuất hiện
   - Review comments được post trên code
   - Summary comment với findings count

### 9.3 Test Chat Commands

Trong PR vừa tạo, comment:
```
@reviewer help
```

Bot sẽ reply với danh sách commands.

---

## Troubleshooting

### Webhook không nhận được

**Symptoms:** Không có activity trong logs khi tạo PR

**Checklist:**
1. Webhook URL đúng và accessible từ internet
2. Webhook secret match giữa GitHub và `.env`
3. App đã được install trên repository
4. Events đã được subscribe (Pull request, Issue comment)

**Debug:**
```bash
# Check webhook deliveries trong GitHub App settings
# Xem response body và status code
```

### Review không được post

**Symptoms:** "Review started" comment xuất hiện nhưng không có review comments

**Checklist:**
1. Celery worker đang chạy
2. Redis connection OK
3. LLM API key valid
4. Đủ credits/quota trên LLM provider

**Debug:**
```bash
# Check Celery logs
celery -A src.workers.celery_app worker --loglevel=debug

# Test LLM connection
python -c "
from src.agents.llm import get_llm
llm = get_llm()
print(llm.invoke('Hello'))
"
```

### Permission Denied errors

**Symptoms:** Error khi post comments lên GitHub

**Checklist:**
1. GitHub App có "Pull requests: Read & Write" permission
2. App đã được reinstall sau khi thay đổi permissions
3. Private key đúng và chưa expired

### Rate Limiting

**Symptoms:** Requests bị reject với 429 status

**Solutions:**
1. Implement exponential backoff (đã có sẵn)
2. Upgrade LLM provider plan
3. Reduce concurrent reviews

---

## Next Steps

Sau khi cài đặt thành công:

1. [Cấu hình chi tiết](configuration.md) - Tùy chỉnh behavior
2. [Automatic Review](auto-review.md) - Hiểu quy trình review
3. [Chat Commands](chat-commands.md) - Sử dụng interactive features
4. [Slack Notifications](slack-notifications.md) - Setup notifications
