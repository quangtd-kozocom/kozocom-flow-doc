# Chat Commands

Bạn có thể tương tác với AI Code Reviewer bằng cách mention `@reviewer` trong comment trên PR.

## Danh sách commands

| Command | Mô tả | Dùng ở đâu |
|---------|-------|------------|
| `@reviewer help` | Xem danh sách commands | Bất kỳ đâu |
| `@reviewer fix this` | Đề xuất code sửa lỗi | Reply vào review comment |
| `@reviewer explain` | Giải thích chi tiết vấn đề | Reply vào review comment |
| `@reviewer generate tests` | Tạo unit tests | Bất kỳ đâu |

---

## `@reviewer help`

Hiển thị danh sách commands có sẵn.

**Cách dùng:** Comment bất kỳ đâu trong PR
```
@reviewer help
```

---

## `@reviewer fix this`

Yêu cầu AI đề xuất code để sửa lỗi.

**Cách dùng:** Reply trực tiếp vào review comment của AI

```
AI comment: "SQL Injection detected..."
    └── Bạn reply: @reviewer fix this
```

**Kết quả:** AI sẽ đưa ra code suggestion có thể apply trực tiếp.

---

## `@reviewer explain`

Yêu cầu AI giải thích chi tiết về vấn đề.

**Cách dùng:** Reply trực tiếp vào review comment của AI

```
AI comment: "Potential null reference..."
    └── Bạn reply: @reviewer explain
```

**Kết quả:** AI sẽ giải thích:
- Tại sao đây là vấn đề
- Hậu quả có thể xảy ra
- Best practices để tránh

---

## `@reviewer generate tests`

Tạo unit tests cho code trong PR.

**Cách dùng:** Comment bất kỳ đâu trong PR

```
@reviewer generate tests
```

**Hoặc cho file cụ thể:**
```
@reviewer tests for src/utils/validator.py
```

**Kết quả:** AI sẽ generate tests bao gồm:
- Happy path tests
- Edge cases
- Error cases

---

## Tips

### Thêm context vào request

```
@reviewer fix this, nhưng giữ backward compatibility
```

```
@reviewer explain bằng tiếng Việt
```

### Follow-up questions

Sau khi AI trả lời, bạn có thể hỏi tiếp trong cùng thread:

```
User: @reviewer explain
AI: [Giải thích về SQL injection...]

User: Cho thêm ví dụ với SQLAlchemy được không?
AI: [Thêm ví dụ...]
```

---

## Cấu hình

### Tắt chat commands

```yaml
# .reviewer.yaml
chat:
  enabled: false
```

### Giới hạn commands

```yaml
chat:
  enabled: true
  allowed_commands:
    - help
    - explain
    # - fix      # Tắt
    # - tests    # Tắt
```

---

**Tiếp theo:** [Cấu hình](configuration.md) - Tùy chỉnh chi tiết
