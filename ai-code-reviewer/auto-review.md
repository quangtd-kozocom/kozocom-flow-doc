# Automatic PR Review

Automatic PR Review là tính năng cốt lõi của AI Code Reviewer. Khi được cài đặt và cấu hình đúng, hệ thống sẽ tự động review mọi Pull Request mà không cần bất kỳ thao tác thủ công nào từ developer.

## Khi nào review được kích hoạt?

Hệ thống tự động bắt đầu review khi một trong các sự kiện sau xảy ra trên repository đã cài đặt GitHub App:

| Event | Mô tả | Khi nào xảy ra |
|-------|-------|----------------|
| `opened` | PR mới được tạo | Developer tạo PR từ branch |
| `synchronize` | PR được cập nhật | Push thêm commits vào PR branch |
| `reopened` | PR được mở lại | PR đã đóng được reopen |

{% hint style="info" %}
Mỗi lần có commits mới được push vào PR, hệ thống sẽ review lại toàn bộ changes trong PR, không chỉ commits mới.
{% endhint %}

---

## Quy trình review chi tiết

### Bước 1: Nhận Webhook Event

Khi một PR event xảy ra, GitHub gửi webhook đến endpoint của AI Code Reviewer.

```
POST /api/v1/webhooks/github
Content-Type: application/json
X-Hub-Signature-256: sha256=<signature>
X-GitHub-Event: pull_request
```

Hệ thống thực hiện các bước xác thực:

1. **Verify Signature**: Kiểm tra HMAC-SHA256 signature để đảm bảo request đến từ GitHub
2. **Parse Event**: Đọc thông tin PR (repo, PR number, action, author, etc.)
3. **Check Conditions**: Kiểm tra các điều kiện auto-review (xem phần [Điều kiện bỏ qua review](#điều-kiện-bỏ-qua-review))

### Bước 2: Acknowledge Review

Ngay sau khi nhận được event hợp lệ, hệ thống post một comment lên PR để thông báo review đang bắt đầu:

```markdown
🔍 **AI Code Reviewer** is analyzing your changes...

This review is powered by AI and may take a few minutes.
```

Comment này giúp developer biết rằng:
- Hệ thống đã nhận được PR
- Review đang được xử lý
- Kết quả sẽ có trong vài phút

### Bước 3: Extract PR Data

Hệ thống fetch thông tin chi tiết về PR từ GitHub API:

**Thông tin được lấy:**

| Data | Mô tả | Sử dụng cho |
|------|-------|-------------|
| PR metadata | Title, description, author, base branch | Filtering, context |
| Changed files | Danh sách files thay đổi | Xác định scope review |
| File diffs | Nội dung thay đổi của từng file | Input cho AI agents |
| File contents | Nội dung đầy đủ của file (nếu cần) | Context cho AI |
| Repository config | File `.reviewer.yaml` nếu có | Custom settings |

**Xử lý diffs:**

```
File: src/utils/validator.py
Status: modified
Additions: 25 lines
Deletions: 10 lines

@@ -15,10 +15,25 @@ def validate_email(email: str) -> bool:
-    if not email:
-        return False
-    return "@" in email
+    if not email or not isinstance(email, str):
+        return False
+    
+    # More robust email validation
+    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
+    return bool(re.match(pattern, email))
```

### Bước 4: Parallel Agent Analysis

Ba AI agents chạy đồng thời để phân tích code:

```
                    ┌─────────────────┐
                    │   PR Changes    │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │  Security   │   │    Logic    │   │    Style    │
    │   Agent     │   │    Agent    │   │    Agent    │
    │             │   │             │   │             │
    │ - SQL Inj.  │   │ - Null ref  │   │ - Naming    │
    │ - XSS       │   │ - Off-by-1  │   │ - Format    │
    │ - Secrets   │   │ - Errors    │   │ - Dead code │
    │ - SSRF      │   │ - Edge case │   │ - Docs      │
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Findings     │
                    │   Collection    │
                    └─────────────────┘
```

**Mỗi agent nhận:**
- Diff của từng file
- Context về file (ngôn ngữ, framework)
- Custom instructions từ `.reviewer.yaml` (nếu có)
- Language setting cho output

**Mỗi agent trả về:**
- Danh sách findings
- Mỗi finding có: file, line, severity, confidence, message, suggestion

### Bước 5: Aggregate Results

Sau khi tất cả agents hoàn thành, hệ thống tổng hợp kết quả:

**Deduplication:**

Khi nhiều agents phát hiện cùng một vấn đề (ví dụ: Security và Logic đều thấy null reference), hệ thống sẽ:
1. Nhóm các findings có cùng file và line number
2. Giữ lại finding có severity cao nhất
3. Merge thông tin từ các findings trùng lặp

**Sorting:**

Findings được sắp xếp theo thứ tự ưu tiên:
1. Severity: `critical` > `warning` > `info` > `suggestion`
2. Confidence: Cao đến thấp
3. File order: Theo thứ tự trong PR

**Filtering:**

- Loại bỏ findings có confidence < threshold (mặc định 0.7)
- Giới hạn số comments per file (mặc định 10)
- Loại bỏ findings trong ignored paths

### Bước 6: Publish Review

Hệ thống post review lên GitHub PR sử dụng GitHub API.

**Review Comments:**

Mỗi finding được post dưới dạng review comment trực tiếp trên dòng code liên quan:

```markdown
**🔴 Critical - Security**

SQL Injection vulnerability detected. User input is directly concatenated 
into SQL query without sanitization.

**Issue:**
The `user_id` parameter is inserted directly into the query string, allowing 
attackers to inject malicious SQL code.

**Recommendation:**
Use parameterized queries or an ORM to prevent SQL injection.

**Suggested fix:**
\`\`\`python
# Instead of:
query = f"SELECT * FROM users WHERE id = {user_id}"

# Use parameterized query:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
\`\`\`

---
*Confidence: 0.95 | Agent: security*
```

**Summary Comment:**

Sau khi post tất cả review comments, hệ thống post một summary comment:

```markdown
## 📊 AI Code Review Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 1 |
| 🟡 Warning | 3 |
| 🔵 Info | 5 |
| ⚪ Suggestion | 2 |

**Total: 11 findings across 4 files**

### Key Issues

1. **SQL Injection** in `src/db/queries.py:45` - Critical
2. **Missing error handling** in `src/api/handlers.py:78` - Warning
3. **Potential null reference** in `src/utils/parser.py:23` - Warning

---

💡 *Reply to any comment with `@reviewer explain` for more details or 
`@reviewer fix this` for a suggested fix.*
```

### Bước 7: Notify (Optional)

Nếu Slack integration được cấu hình, hệ thống gửi notification:

```
🔍 Code Review Completed

Repository: myorg/myrepo
PR #123: Add user authentication

Results:
• 1 Critical
• 3 Warnings  
• 5 Info
• 2 Suggestions

View PR: https://github.com/myorg/myrepo/pull/123
```

---

## Điều kiện bỏ qua review

Hệ thống sẽ **không** review PR trong các trường hợp sau:

### 1. Skip Keywords trong PR Title

Nếu title của PR chứa các keywords được cấu hình, review sẽ bị bỏ qua.

**Mặc định:**
- `[WIP]` - Work in Progress
- `[SKIP REVIEW]` - Explicit skip
- `[NO REVIEW]` - Explicit skip
- `[DRAFT]` - Draft PR

**Ví dụ:**
```
[WIP] Add new authentication flow     ← Skipped
[SKIP REVIEW] Quick typo fix          ← Skipped
Add user registration feature         ← Reviewed
```

**Cấu hình trong `.reviewer.yaml`:**
```yaml
reviews:
  auto_review:
    skip_keywords:
      - "[WIP]"
      - "[SKIP REVIEW]"
      - "[HOTFIX]"  # Custom keyword
```

### 2. Ignored Authors

PR từ các authors được cấu hình sẽ bị bỏ qua. Thường dùng cho bots.

**Mặc định:**
- `dependabot[bot]`
- `renovate[bot]`
- `github-actions[bot]`

**Cấu hình:**
```yaml
reviews:
  auto_review:
    ignore_authors:
      - dependabot[bot]
      - renovate[bot]
      - my-deploy-bot
```

### 3. Base Branch Filter

Chỉ review PR merge vào các branches được chỉ định.

**Ví dụ:** Chỉ review PR vào `main` và `develop`:
```yaml
reviews:
  auto_review:
    base_branches:
      - main
      - develop
```

PR merge vào `feature/xyz` sẽ không được review.

### 4. Draft PRs

Mặc định, draft PRs không được review. Có thể bật nếu cần:

```yaml
reviews:
  auto_review:
    drafts: true  # Review cả draft PRs
```

### 5. Disabled Auto-review

Có thể tắt hoàn toàn auto-review:

```yaml
reviews:
  auto_review:
    enabled: false
```

Khi tắt, vẫn có thể trigger review thủ công qua chat commands.

---

## Ignored Files và Paths

Một số files không nên được review vì không có giá trị hoặc gây noise.

### Default Ignored Patterns

```yaml
ignore:
  # Lock files
  - "**/package-lock.json"
  - "**/yarn.lock"
  - "**/pnpm-lock.yaml"
  - "**/Pipfile.lock"
  - "**/poetry.lock"
  - "**/composer.lock"
  
  # Generated files
  - "**/migrations/**"
  - "**/*.generated.*"
  - "**/*.min.js"
  - "**/*.min.css"
  - "**/dist/**"
  - "**/build/**"
  
  # Dependencies
  - "**/node_modules/**"
  - "**/vendor/**"
  - "**/.venv/**"
  
  # IDE/Editor
  - "**/.idea/**"
  - "**/.vscode/**"
  
  # Assets
  - "**/*.png"
  - "**/*.jpg"
  - "**/*.svg"
  - "**/*.ico"
```

### Custom Ignore Patterns

Thêm patterns trong `.reviewer.yaml`:

```yaml
ignore:
  # Ignore test fixtures
  - "**/fixtures/**"
  - "**/__snapshots__/**"
  
  # Ignore specific directories
  - "legacy/**"
  - "deprecated/**"
  
  # Ignore specific file types
  - "**/*.pb.go"  # Generated protobuf
  - "**/*.graphql.ts"  # Generated GraphQL types
```

---

## Path-specific Instructions

Có thể cung cấp instructions đặc biệt cho các paths cụ thể để AI tập trung vào những vấn đề quan trọng.

### Ví dụ cấu hình

```yaml
reviews:
  path_instructions:
    # API endpoints - focus on security
    - path: "src/api/**/*.py"
      instructions: |
        Focus on:
        - Input validation and sanitization
        - Authentication and authorization checks
        - Rate limiting considerations
        - Proper error responses (don't leak internal details)
    
    # Database models - focus on data integrity
    - path: "src/models/**/*.py"
      instructions: |
        Focus on:
        - Proper field validation
        - Index usage for query performance
        - N+1 query potential
        - Data migration safety
    
    # Frontend components - focus on UX and accessibility
    - path: "src/components/**/*.tsx"
      instructions: |
        Focus on:
        - Accessibility (ARIA labels, keyboard navigation)
        - Performance (unnecessary re-renders)
        - Error states and loading states
        - Responsive design considerations
    
    # Tests - focus on coverage and assertions
    - path: "tests/**/*.py"
      instructions: |
        Focus on:
        - Test coverage completeness
        - Meaningful assertions
        - Edge case coverage
        - Test isolation (no shared state)
```

### Glob Pattern Syntax

| Pattern | Matches |
|---------|---------|
| `*` | Any characters except `/` |
| `**` | Any characters including `/` |
| `?` | Single character |
| `[abc]` | Character class |
| `{a,b}` | Alternatives |

**Ví dụ:**
- `src/**/*.py` - Tất cả Python files trong `src/`
- `*.{js,ts}` - JavaScript và TypeScript files
- `src/api/v[12]/**` - Files trong `src/api/v1/` hoặc `src/api/v2/`

---

## Review Profiles

Profiles cho phép điều chỉnh độ nghiêm ngặt của review.

### Available Profiles

| Profile | Mô tả | Use case |
|---------|-------|----------|
| `chill` | Ít comments, chỉ vấn đề quan trọng | Prototypes, hackathons |
| `default` | Cân bằng giữa thoroughness và noise | Production code |
| `strict` | Nhiều comments, bắt cả minor issues | Critical systems, security-sensitive |

### Profile Behaviors

**`chill` Profile:**
- Chỉ report `critical` và `warning`
- Confidence threshold: 0.85
- Max 5 comments per file
- Bỏ qua style issues

**`default` Profile:**
- Report tất cả severities
- Confidence threshold: 0.7
- Max 10 comments per file
- Balanced style checking

**`strict` Profile:**
- Report tất cả severities
- Confidence threshold: 0.6
- Max 20 comments per file
- Strict style enforcement
- Additional checks enabled

### Cấu hình Profile

```yaml
reviews:
  profile: "strict"  # chill | default | strict
```

---

## Xử lý Large PRs

Với PRs có nhiều files hoặc changes lớn, hệ thống áp dụng các chiến lược sau:

### File Prioritization

Khi PR có quá nhiều files, hệ thống ưu tiên:
1. Files có nhiều changes nhất
2. Files trong critical paths (API, auth, database)
3. Files mới (added) over files modified
4. Source files over test files

### Chunking

Với files rất lớn, diff được chia thành chunks và xử lý tuần tự để tránh token limit của LLM.

### Timeout Handling

- Mỗi review task có timeout 5 phút
- Nếu timeout, partial results vẫn được post
- Summary sẽ ghi chú về incomplete review

---

## Retry và Error Handling

### Automatic Retries

Khi gặp lỗi transient (network, rate limit), hệ thống tự động retry:

- Max retries: 3
- Backoff: Exponential (1s, 2s, 4s)
- Retryable errors: Network timeout, 5xx responses, rate limits

### Error Notifications

Khi review fail sau tất cả retries, hệ thống:
1. Post comment thông báo lỗi trên PR
2. Log error details cho debugging
3. Gửi Slack notification (nếu configured)

```markdown
⚠️ **AI Code Review Failed**

The automated review could not be completed due to an error.
Our team has been notified and will investigate.

You can try again by pushing a new commit or commenting `@reviewer review`.

Error ID: `abc123` (for support reference)
```
