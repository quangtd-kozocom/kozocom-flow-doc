# Cấu hình

Tùy chỉnh AI Code Reviewer cho repository của bạn bằng file `.reviewer.yaml`.

## Tạo file cấu hình

Tạo file `.reviewer.yaml` ở **root** của repository:

```
your-repo/
├── .reviewer.yaml    ← Tạo file này
├── src/
└── ...
```

## Cấu hình cơ bản

```yaml
# Ngôn ngữ cho review comments
language: "vi"  # vi, en, ja

# Độ nghiêm ngặt
reviews:
  profile: "default"  # chill, default, strict
```

## Cấu hình đầy đủ

```yaml
# ======================
# NGÔN NGỮ
# ======================
language: "vi"

# ======================
# REVIEW SETTINGS
# ======================
reviews:
  # Độ nghiêm ngặt: chill | default | strict
  profile: "default"
  
  # Chọn agents chạy
  agents:
    - security
    - logic
    - style
  
  # Ngưỡng confidence (0.0 - 1.0)
  # Cao hơn = ít comments hơn nhưng chính xác hơn
  confidence_threshold: 0.7
  
  # Số comments tối đa mỗi file
  max_comments_per_file: 10
  
  # Auto-review settings
  auto_review:
    enabled: true
    drafts: false  # Review draft PRs?
    
    # Bỏ qua PR có title chứa
    skip_keywords:
      - "[WIP]"
      - "[SKIP REVIEW]"
    
    # Chỉ review PR vào branches này
    base_branches:
      - main
      - develop
    
    # Bỏ qua PR từ authors này
    ignore_authors:
      - dependabot[bot]
      - renovate[bot]

# ======================
# IGNORE FILES
# ======================
ignore:
  - "**/migrations/**"
  - "**/*.min.js"
  - "**/package-lock.json"
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/*.generated.*"

# ======================
# CHAT COMMANDS
# ======================
chat:
  enabled: true
  allowed_commands:
    - help
    - fix
    - explain
    - tests
```

---

## Chi tiết các options

### `language`

Ngôn ngữ cho review comments.

| Giá trị | Ngôn ngữ |
|---------|----------|
| `en` | English |
| `vi` | Tiếng Việt |
| `ja` | 日本語 |

### `reviews.profile`

Độ nghiêm ngặt của review.

| Profile | Mô tả |
|---------|-------|
| `chill` | Ít comments, chỉ vấn đề quan trọng |
| `default` | Cân bằng |
| `strict` | Nhiều comments, bắt cả lỗi nhỏ |

### `reviews.agents`

Chọn agents sẽ chạy:

- `security` - Lỗ hổng bảo mật
- `logic` - Lỗi logic, edge cases
- `style` - Code style, conventions

**Ví dụ:** Chỉ chạy security check:
```yaml
reviews:
  agents:
    - security
```

### `ignore`

Glob patterns cho files không cần review.

**Patterns phổ biến:**

```yaml
ignore:
  # Lock files
  - "**/package-lock.json"
  - "**/yarn.lock"
  - "**/pnpm-lock.yaml"
  
  # Generated
  - "**/migrations/**"
  - "**/*.generated.*"
  - "**/*.min.js"
  
  # Build output
  - "**/dist/**"
  - "**/build/**"
  
  # Dependencies
  - "**/node_modules/**"
  - "**/vendor/**"
```

---

## Ví dụ theo use case

### Dự án mới / Prototype

```yaml
language: "vi"
reviews:
  profile: "chill"
  agents:
    - security  # Chỉ check security
```

### Production codebase

```yaml
language: "vi"
reviews:
  profile: "strict"
  confidence_threshold: 0.6
  max_comments_per_file: 20
```

### Monorepo

```yaml
reviews:
  path_instructions:
    - path: "packages/api/**"
      instructions: "Focus on API security"
    - path: "packages/web/**"
      instructions: "Focus on React best practices"
```

---

**Tiếp theo:** [Slack Notifications](slack-notifications.md)
