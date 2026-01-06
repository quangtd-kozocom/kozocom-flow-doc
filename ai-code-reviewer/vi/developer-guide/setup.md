# Cài đặt cho Developers

Hướng dẫn clone và chạy AI Code Reviewer locally để phát triển.

## Yêu cầu

- Python 3.13+
- Redis
- GitHub App credentials
- API key từ OpenRouter/OpenAI/Anthropic

## Bước 1: Clone project

```bash
git clone https://github.com/your-org/ai-code-review-kozocom.git
cd ai-code-review-kozocom
```

## Bước 2: Cài đặt dependencies

{% tabs %}
{% tab title="uv (Recommended)" %}
```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Activate
source .venv/bin/activate
```
{% endtab %}

{% tab title="pip" %}
```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
```
{% endtab %}
{% endtabs %}

## Bước 3: Cài đặt Redis

{% tabs %}
{% tab title="macOS" %}
```bash
brew install redis
brew services start redis
```
{% endtab %}

{% tab title="Ubuntu" %}
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
```
{% endtab %}

{% tab title="Docker" %}
```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```
{% endtab %}
{% endtabs %}

Verify:
```bash
redis-cli ping
# Expected: PONG
```

## Bước 4: Tạo GitHub App

1. Vào [github.com/settings/apps](https://github.com/settings/apps) → **New GitHub App**

2. Điền thông tin:
   - **Name:** `AI Code Reviewer Dev`
   - **Homepage URL:** `http://localhost:8000`
   - **Webhook URL:** `https://your-ngrok-url/api/v1/webhooks/github`
   - **Webhook secret:** Generate random string

3. Permissions:
   | Permission | Access |
   |------------|--------|
   | Contents | Read |
   | Metadata | Read |
   | Pull requests | Read & Write |

4. Subscribe events:
   - Pull request
   - Issue comment

5. **Generate private key** → Download file `.pem`

6. **Install App** vào repository test

## Bước 5: Cấu hình environment

```bash
cp .env.example .env
```

Sửa file `.env`:

```bash
# GitHub App
GITHUB_APP_ID=123456
GITHUB_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
GITHUB_WEBHOOK_SECRET=your-secret

# LLM (chọn 1)
OPENROUTER_API_KEY=sk-or-v1-xxx
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# Redis
REDIS_URL=redis://localhost:6379
```

{% hint style="info" %}
**Private key:** Convert file .pem thành single line:
```bash
cat private-key.pem | awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}'
```
{% endhint %}

## Bước 6: Chạy services

**Terminal 1 - API:**
```bash
make dev
```

**Terminal 2 - Worker:**
```bash
make worker
```

**Hoặc dùng script:**
```bash
./scripts/start.sh --tmux
```

## Bước 7: Expose webhook (ngrok)

```bash
ngrok http 8000
```

Copy URL (vd: `https://abc123.ngrok.io`) và update vào GitHub App webhook URL.

## Verify

1. Tạo PR trong repo đã install app
2. Đợi 1-2 phút
3. Kiểm tra comments trên PR

---

## Make commands

| Command | Mô tả |
|---------|-------|
| `make dev` | Chạy API với hot-reload |
| `make worker` | Chạy Celery worker |
| `make test` | Chạy tests |
| `make lint` | Check code style |
| `make format` | Auto-format code |

---

**Tiếp theo:** [Kiến trúc hệ thống](architecture.md)
