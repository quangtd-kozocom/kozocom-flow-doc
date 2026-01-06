# Auto Review

Khi bạn tạo hoặc cập nhật Pull Request, AI Code Reviewer sẽ tự động phân tích code và đưa ra nhận xét.

## Cách hoạt động

```
Bạn tạo PR  →  AI nhận webhook  →  3 Agents phân tích  →  Comments trên PR
```

**3 AI Agents chuyên biệt:**

| Agent | Kiểm tra |
|-------|----------|
| 🔒 **Security** | SQL Injection, XSS, hardcoded secrets, SSRF |
| 🧠 **Logic** | Null reference, off-by-one, missing error handling |
| 🎨 **Style** | Naming conventions, dead code, missing docs |

## Khi nào review được kích hoạt?

- ✅ PR mới được tạo
- ✅ Push thêm commits vào PR
- ✅ Reopen PR đã đóng

## Bỏ qua review

### Thêm keyword vào title PR

```
[WIP] Đang làm dở          ← Bỏ qua
[SKIP REVIEW] Hotfix       ← Bỏ qua
Add new feature            ← Được review
```

### Cấu hình trong `.reviewer.yaml`

```yaml
reviews:
  auto_review:
    # Tắt hoàn toàn
    enabled: false
    
    # Hoặc tùy chỉnh
    skip_keywords:
      - "[WIP]"
      - "[HOTFIX]"
    
    # Bỏ qua PR từ bots
    ignore_authors:
      - dependabot[bot]
```

## Hiểu kết quả review

### Mức độ nghiêm trọng

| Mức độ | Ý nghĩa | Hành động |
|--------|---------|-----------|
| 🔴 **Critical** | Lỗi bảo mật nghiêm trọng | Phải sửa |
| 🟡 **Warning** | Bug tiềm ẩn | Nên sửa |
| 🔵 **Info** | Gợi ý cải thiện | Cân nhắc |
| ⚪ **Suggestion** | Style/optimization | Tùy chọn |

### Ví dụ comment

```markdown
🔴 **Critical - Security**

SQL Injection detected. User input is directly used in query.

**Vấn đề:** Biến `user_id` được đưa trực tiếp vào query.

**Gợi ý sửa:**
```python
# Thay vì:
query = f"SELECT * FROM users WHERE id = {user_id}"

# Dùng parameterized query:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```
```

## Ignore files

Một số files không cần review:

```yaml
# .reviewer.yaml
ignore:
  - "**/migrations/**"
  - "**/*.min.js"
  - "**/package-lock.json"
  - "**/node_modules/**"
```

---

**Tiếp theo:** [Chat Commands](chat-commands.md) - Tương tác với AI qua comments
