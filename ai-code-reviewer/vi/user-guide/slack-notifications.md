# Slack Notifications

Nhận thông báo qua Slack khi AI hoàn thành review PR.

## Yêu cầu

- Slack workspace
- Quyền tạo Slack App (hoặc nhờ admin)

## Cài đặt

### Bước 1: Tạo Slack App

1. Vào [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. Đặt tên (vd: "AI Code Reviewer")
4. Chọn workspace

### Bước 2: Cấu hình permissions

1. Vào **OAuth & Permissions**
2. Thêm **Bot Token Scopes**:
   - `chat:write`
   - `chat:write.public`

### Bước 3: Install App

1. Click **Install to Workspace**
2. Copy **Bot User OAuth Token** (bắt đầu bằng `xoxb-`)

### Bước 4: Invite bot vào channel

Trong Slack, vào channel muốn nhận thông báo:
```
/invite @AI Code Reviewer
```

### Bước 5: Cấu hình server

Thêm vào environment variables của server:

```bash
SLACK_BOT_TOKEN=xoxb-your-token
SLACK_CHANNEL=#pr-reviews
```

---

## Format thông báo

### Review hoàn thành

```
🔍 Code Review Completed

Repository: myorg/myrepo
PR #123: Add user authentication

📊 Results:
• 🔴 Critical: 1
• 🟡 Warning: 3
• 🔵 Info: 5

View PR: https://github.com/...
```

### Không có vấn đề

```
✅ Code Review Completed

Repository: myorg/myrepo  
PR #456: Fix typo

🎉 No issues found!
```

---

## Troubleshooting

### Không nhận được thông báo

1. Kiểm tra bot token còn valid
2. Kiểm tra bot đã được invite vào channel
3. Kiểm tra channel name/ID đúng

### Test connection

```bash
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channel":"#pr-reviews","text":"Test message"}'
```

---

**Quay lại:** [Cấu hình](configuration.md)
