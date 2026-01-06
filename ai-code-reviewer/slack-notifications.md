# Slack Notifications

Slack Notifications cho phép team nhận thông báo real-time khi AI Code Reviewer hoàn thành review một Pull Request. Tính năng này giúp team không bỏ lỡ các review quan trọng và phản hồi nhanh hơn.

## Tổng quan

Khi một PR được review xong, hệ thống sẽ gửi notification đến Slack channel được cấu hình với thông tin:

- Repository và PR number
- Tên PR và author
- Tóm tắt kết quả review (số lượng findings theo severity)
- Link trực tiếp đến PR

---

## Cài đặt Slack App

### Bước 1: Tạo Slack App

1. Truy cập [Slack API Apps](https://api.slack.com/apps)
2. Click **"Create New App"**
3. Chọn **"From scratch"**
4. Đặt tên app (ví dụ: "AI Code Reviewer")
5. Chọn workspace muốn cài đặt

### Bước 2: Cấu hình Bot Permissions

1. Trong app settings, vào **"OAuth & Permissions"**
2. Scroll xuống **"Scopes"** → **"Bot Token Scopes"**
3. Thêm các permissions sau:

| Scope | Mô tả | Bắt buộc |
|-------|-------|----------|
| `chat:write` | Gửi messages đến channels | Yes |
| `chat:write.public` | Gửi messages đến public channels mà bot chưa join | Yes |
| `channels:read` | Đọc thông tin channels | Optional |

### Bước 3: Install App to Workspace

1. Vào **"OAuth & Permissions"**
2. Click **"Install to Workspace"**
3. Review permissions và click **"Allow"**
4. Copy **"Bot User OAuth Token"** (bắt đầu bằng `xoxb-`)

### Bước 4: Invite Bot vào Channel

Trong Slack, vào channel muốn nhận notifications:

```
/invite @AI Code Reviewer
```

Hoặc mention bot trong channel để Slack tự động invite.

---

## Cấu hình Environment Variables

Thêm các biến sau vào file `.env`:

```bash
# Slack Integration
SLACK_BOT_TOKEN=xoxb-your-bot-token-here
SLACK_CHANNEL=#pr-reviews
```

### Chi tiết các biến

| Variable | Required | Default | Mô tả |
|----------|----------|---------|-------|
| `SLACK_BOT_TOKEN` | Yes | - | Bot User OAuth Token từ Slack App |
| `SLACK_CHANNEL` | No | `#pr-reviews` | Channel nhận notifications |

### Channel Format

Channel có thể được chỉ định theo nhiều cách:

```bash
# Bằng tên (có hoặc không có #)
SLACK_CHANNEL=#pr-reviews
SLACK_CHANNEL=pr-reviews

# Bằng Channel ID (recommended cho private channels)
SLACK_CHANNEL=C01234ABCDE
```

{% hint style="info" %}
Sử dụng Channel ID thay vì tên channel để tránh issues khi channel được rename.
{% endhint %}

**Cách lấy Channel ID:**
1. Mở channel trong Slack
2. Click vào tên channel ở header
3. Scroll xuống cuối popup, Channel ID hiển thị ở đó

---

## Format Notification

### Review Completed - Có Findings

```
┌─────────────────────────────────────────────────────────────┐
│  🔍 Code Review Completed                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Repository: myorg/myrepo                                   │
│  PR #123: Add user authentication feature                   │
│  Author: @johndoe                                           │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  📊 Review Summary                                          │
│                                                             │
│  🔴 Critical    2                                           │
│  🟡 Warning     5                                           │
│  🔵 Info        8                                           │
│  ⚪ Suggestion  3                                           │
│                                                             │
│  Total: 18 findings across 6 files                          │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  ⚠️ Critical Issues Found:                                  │
│  • SQL Injection in src/db/queries.py:45                    │
│  • Hardcoded secret in src/config.py:12                     │
│                                                             │
│  [View Pull Request]                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Review Completed - Không có Findings

```
┌─────────────────────────────────────────────────────────────┐
│  ✅ Code Review Completed                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Repository: myorg/myrepo                                   │
│  PR #456: Fix typo in README                                │
│  Author: @janedoe                                           │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  🎉 No issues found!                                        │
│                                                             │
│  The code changes look good. No security, logic, or style   │
│  issues were detected.                                      │
│                                                             │
│  [View Pull Request]                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Review Failed

```
┌─────────────────────────────────────────────────────────────┐
│  ❌ Code Review Failed                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Repository: myorg/myrepo                                   │
│  PR #789: Large refactoring                                 │
│  Author: @developer                                         │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  The automated review could not be completed.               │
│                                                             │
│  Error: Review timeout - PR has too many changes            │
│  Error ID: abc123xyz                                        │
│                                                             │
│  Please try again or contact support if the issue persists. │
│                                                             │
│  [View Pull Request]                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Notification Triggers

Notifications được gửi trong các trường hợp sau:

| Event | Notification | Điều kiện |
|-------|--------------|-----------|
| Review completed | Yes | Luôn gửi khi review hoàn thành |
| Review failed | Yes | Khi review gặp lỗi sau tất cả retries |
| Critical findings | Yes | Highlight trong notification |
| No findings | Yes | Thông báo "clean" review |

### Không gửi notification khi:

- PR bị skip do skip keywords
- PR từ ignored authors
- Slack integration chưa được cấu hình
- Bot không có quyền gửi message đến channel

---

## Advanced Configuration

### Multiple Channels

Hiện tại hệ thống chỉ hỗ trợ một channel. Để gửi đến nhiều channels, bạn có thể:

1. **Sử dụng Slack Workflow**: Tạo workflow forward messages từ channel chính đến các channels khác
2. **Channel với nhiều members**: Invite tất cả người cần nhận notification vào cùng channel

### Channel per Repository

Tính năng này đang được phát triển. Trong tương lai sẽ hỗ trợ:

```yaml
# .reviewer.yaml (planned)
notifications:
  slack:
    channel: "#myrepo-reviews"
```

### Filtering Notifications

Để giảm noise, bạn có thể cấu hình chỉ gửi notification khi có critical findings:

```yaml
# .reviewer.yaml (planned)
notifications:
  slack:
    only_critical: true
```

---

## Troubleshooting

### Notification không được gửi

**Kiểm tra:**

1. **Bot token hợp lệ:**
   ```bash
   curl -X POST https://slack.com/api/auth.test \
     -H "Authorization: Bearer $SLACK_BOT_TOKEN"
   ```
   
   Response thành công:
   ```json
   {
     "ok": true,
     "url": "https://yourworkspace.slack.com/",
     "team": "Your Workspace",
     "user": "ai-code-reviewer",
     "team_id": "T01234567",
     "user_id": "U01234567"
   }
   ```

2. **Bot có quyền gửi message:**
   - Kiểm tra `chat:write` scope trong Slack App settings
   - Reinstall app nếu vừa thêm scope mới

3. **Bot đã được invite vào channel:**
   - Private channels: Bot phải được invite
   - Public channels: Cần `chat:write.public` scope hoặc invite bot

4. **Channel name/ID đúng:**
   - Kiểm tra typo trong `SLACK_CHANNEL`
   - Thử dùng Channel ID thay vì tên

### Message format bị lỗi

**Nguyên nhân:** Slack API thay đổi hoặc special characters trong PR title

**Giải pháp:**
- Kiểm tra logs để xem error message
- Report issue nếu format bị broken

### Rate limiting

Slack có rate limits cho API calls. Nếu team có nhiều PRs:

- Limit: ~1 message/second per channel
- Hệ thống tự động queue và retry
- Notifications có thể bị delay vài giây

---

## Security Considerations

### Token Security

- **Không commit** `SLACK_BOT_TOKEN` vào repository
- Sử dụng environment variables hoặc secrets manager
- Rotate token định kỳ (6 tháng/lần)

### Channel Access

- Chỉ invite bot vào channels cần thiết
- Sử dụng private channel nếu review chứa sensitive information
- Review bot permissions định kỳ

### Data in Notifications

Notifications chỉ chứa:
- Repository name (public info)
- PR number và title
- Author username
- Summary counts (không có code content)

**Không bao gồm:**
- Actual code snippets
- Detailed finding descriptions
- File contents

---

## Disable Notifications

### Tạm thời disable

Comment out hoặc remove `SLACK_BOT_TOKEN` trong `.env`:

```bash
# SLACK_BOT_TOKEN=xoxb-...
# SLACK_CHANNEL=#pr-reviews
```

### Disable cho specific repository

Tính năng này đang được phát triển. Workaround hiện tại:
- Không install GitHub App trên repository đó
- Hoặc sử dụng skip keywords trong PR titles
