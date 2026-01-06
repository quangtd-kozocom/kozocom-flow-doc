# Chat Commands

Chat Commands cho phép developers tương tác trực tiếp với AI Code Reviewer thông qua comments trên Pull Request. Thay vì chỉ nhận review một chiều, bạn có thể yêu cầu giải thích, đề xuất fix, hoặc generate tests.

## Cách sử dụng

Để sử dụng chat commands, bạn chỉ cần mention `@reviewer` trong comment trên PR, theo sau là command muốn thực hiện.

```
@reviewer <command> [arguments]
```

**Lưu ý quan trọng:**
- Commands chỉ hoạt động trong Pull Request comments
- Một số commands cần được reply trực tiếp vào review comment
- Bot sẽ phản hồi trong cùng thread với comment của bạn

---

## Danh sách Commands

### `@reviewer help`

Hiển thị danh sách tất cả commands có sẵn và cách sử dụng.

**Cách dùng:**
```
@reviewer help
```

**Nơi sử dụng:** Bất kỳ comment nào trong PR

**Response:**
```markdown
## 🤖 AI Code Reviewer - Available Commands

| Command | Description | Usage |
|---------|-------------|-------|
| `help` | Show this help message | `@reviewer help` |
| `fix this` | Generate a code fix | Reply to a review comment |
| `explain` | Explain the issue in detail | Reply to a review comment |
| `generate tests` | Generate unit tests | Comment anywhere |
| `tests for <file>` | Generate tests for specific file | Comment anywhere |

### Examples

**Get a fix suggestion:**
> Reply to a review comment with: `@reviewer fix this`

**Generate tests:**
> Comment: `@reviewer generate tests`
> Or: `@reviewer tests for src/utils/validator.py`
```

---

### `@reviewer fix this`

Yêu cầu AI đề xuất code fix cho một vấn đề được phát hiện trong review.

**Cách dùng:**
```
@reviewer fix this
```

**Nơi sử dụng:** Reply trực tiếp vào một review comment từ AI Code Reviewer

**Cách hoạt động:**

1. AI đọc review comment gốc để hiểu vấn đề
2. Fetch context của file và surrounding code
3. Phân tích root cause của vấn đề
4. Generate code fix phù hợp với codebase style
5. Post response với suggested fix

**Ví dụ sử dụng:**

Giả sử AI Code Reviewer đã post review comment sau:

```markdown
**🟡 Warning - Logic**

Potential null reference. The `user` object may be None but is accessed 
without null check.

File: src/api/handlers.py
Line: 45
```

Bạn reply vào comment này:
```
@reviewer fix this
```

AI sẽ response:
```markdown
## 🔧 Suggested Fix

Here's how to fix the null reference issue:

**Current code:**
```python
def get_user_name(user_id: int) -> str:
    user = db.get_user(user_id)
    return user.name  # ← Potential None access
```

**Fixed code:**
```python
def get_user_name(user_id: int) -> str:
    user = db.get_user(user_id)
    if user is None:
        raise UserNotFoundError(f"User {user_id} not found")
    return user.name
```

**Explanation:**
- Added explicit null check before accessing `user.name`
- Raises a descriptive exception instead of causing AttributeError
- Consider using `Optional[str]` return type if returning None is acceptable

**Alternative approach:**
```python
def get_user_name(user_id: int) -> Optional[str]:
    user = db.get_user(user_id)
    return user.name if user else None
```

---
*Click "Commit suggestion" to apply this fix directly.*
```

**Tips:**
- Bạn có thể thêm context vào request: `@reviewer fix this, but keep backward compatibility`
- AI sẽ cố gắng match code style của project
- Suggested fix có thể được commit trực tiếp qua GitHub UI

---

### `@reviewer explain`

Yêu cầu AI giải thích chi tiết về một vấn đề được phát hiện.

**Cách dùng:**
```
@reviewer explain
```

**Nơi sử dụng:** Reply trực tiếp vào một review comment từ AI Code Reviewer

**Cách hoạt động:**

1. AI đọc review comment gốc
2. Phân tích vấn đề trong context rộng hơn
3. Giải thích tại sao đây là vấn đề
4. Cung cấp thông tin về potential impact
5. Đưa ra best practices và references

**Ví dụ sử dụng:**

Giả sử AI Code Reviewer đã post review comment:

```markdown
**🔴 Critical - Security**

SQL Injection vulnerability detected.

File: src/db/queries.py
Line: 23
```

Bạn reply:
```
@reviewer explain
```

AI sẽ response:
```markdown
## 📚 Detailed Explanation: SQL Injection

### What is SQL Injection?

SQL Injection là một trong những lỗ hổng bảo mật phổ biến và nguy hiểm nhất 
(OWASP Top 10 #3). Nó xảy ra khi user input được đưa trực tiếp vào SQL query 
mà không được sanitize.

### Why is this code vulnerable?

```python
# Line 23 in src/db/queries.py
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)
```

Vấn đề: `username` được interpolate trực tiếp vào query string.

### Attack Scenario

Attacker có thể input:
```
username = "admin' OR '1'='1' --"
```

Query trở thành:
```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1' --'
```

Kết quả: Trả về TẤT CẢ users trong database!

### Potential Impact

| Impact | Severity | Description |
|--------|----------|-------------|
| Data Breach | Critical | Attacker có thể đọc toàn bộ database |
| Data Modification | Critical | Có thể UPDATE/DELETE data |
| Authentication Bypass | Critical | Bypass login không cần password |
| Remote Code Execution | Critical | Một số DB cho phép execute OS commands |

### Real-world Examples

- **2017 Equifax Breach**: 147 million records exposed
- **2011 Sony PlayStation**: 77 million accounts compromised
- **2008 Heartland Payment**: 134 million credit cards stolen

### How to Fix

**Option 1: Parameterized Queries (Recommended)**
```python
cursor.execute(
    "SELECT * FROM users WHERE username = %s", 
    (username,)
)
```

**Option 2: ORM (Best)**
```python
user = User.query.filter_by(username=username).first()
```

**Option 3: Input Validation (Additional layer)**
```python
if not re.match(r'^[a-zA-Z0-9_]+$', username):
    raise InvalidInputError("Invalid username format")
```

### References

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- [Bobby Tables](https://bobby-tables.com/) - Guide to preventing SQL injection

---
*Reply with `@reviewer fix this` to get a code fix suggestion.*
```

**Tips:**
- Explanation được viết bằng ngôn ngữ đã cấu hình trong `.reviewer.yaml`
- Bao gồm real-world examples và references
- Có thể hỏi thêm: `@reviewer explain more about the impact`

---

### `@reviewer generate tests`

Yêu cầu AI generate unit tests cho tất cả files thay đổi trong PR.

**Cách dùng:**
```
@reviewer generate tests
```

**Nơi sử dụng:** Bất kỳ comment nào trong PR (không cần reply vào review comment)

**Cách hoạt động:**

1. AI fetch danh sách files thay đổi trong PR
2. Phân tích code để hiểu functionality
3. Identify test cases cần cover
4. Generate test code phù hợp với testing framework của project
5. Post tests cho từng file

**Ví dụ sử dụng:**

Comment trong PR:
```
@reviewer generate tests
```

AI sẽ response:
```markdown
## 🧪 Generated Unit Tests

I've analyzed the changes in this PR and generated tests for the modified files.

### Tests for `src/utils/validator.py`

```python
# tests/test_validator.py

import pytest
from src.utils.validator import validate_email, validate_phone, validate_url


class TestValidateEmail:
    """Tests for validate_email function."""
    
    def test_valid_email_simple(self):
        """Test with a simple valid email."""
        assert validate_email("user@example.com") is True
    
    def test_valid_email_with_subdomain(self):
        """Test with subdomain in email."""
        assert validate_email("user@mail.example.com") is True
    
    def test_valid_email_with_plus(self):
        """Test with plus addressing."""
        assert validate_email("user+tag@example.com") is True
    
    def test_invalid_email_no_at(self):
        """Test email without @ symbol."""
        assert validate_email("userexample.com") is False
    
    def test_invalid_email_no_domain(self):
        """Test email without domain."""
        assert validate_email("user@") is False
    
    def test_invalid_email_empty(self):
        """Test with empty string."""
        assert validate_email("") is False
    
    def test_invalid_email_none(self):
        """Test with None input."""
        assert validate_email(None) is False
    
    def test_invalid_email_spaces(self):
        """Test email with spaces."""
        assert validate_email("user @example.com") is False


class TestValidatePhone:
    """Tests for validate_phone function."""
    
    def test_valid_phone_10_digits(self):
        """Test with 10-digit phone number."""
        assert validate_phone("0901234567") is True
    
    def test_valid_phone_with_country_code(self):
        """Test with country code."""
        assert validate_phone("+84901234567") is True
    
    def test_invalid_phone_too_short(self):
        """Test with too few digits."""
        assert validate_phone("12345") is False
    
    def test_invalid_phone_letters(self):
        """Test with letters in phone."""
        assert validate_phone("090ABC4567") is False


class TestValidateUrl:
    """Tests for validate_url function."""
    
    def test_valid_url_https(self):
        """Test with HTTPS URL."""
        assert validate_url("https://example.com") is True
    
    def test_valid_url_with_path(self):
        """Test URL with path."""
        assert validate_url("https://example.com/path/to/page") is True
    
    def test_valid_url_with_query(self):
        """Test URL with query parameters."""
        assert validate_url("https://example.com?foo=bar") is True
    
    def test_invalid_url_no_protocol(self):
        """Test URL without protocol."""
        assert validate_url("example.com") is False
    
    def test_invalid_url_javascript(self):
        """Test javascript: URL (security)."""
        assert validate_url("javascript:alert(1)") is False
```

### Tests for `src/api/handlers.py`

```python
# tests/test_handlers.py

import pytest
from unittest.mock import Mock, patch
from src.api.handlers import get_user, create_user, update_user


class TestGetUser:
    """Tests for get_user handler."""
    
    @patch('src.api.handlers.db')
    def test_get_user_exists(self, mock_db):
        """Test getting an existing user."""
        mock_db.get_user.return_value = Mock(
            id=1, 
            name="John Doe", 
            email="john@example.com"
        )
        
        result = get_user(user_id=1)
        
        assert result["id"] == 1
        assert result["name"] == "John Doe"
        mock_db.get_user.assert_called_once_with(1)
    
    @patch('src.api.handlers.db')
    def test_get_user_not_found(self, mock_db):
        """Test getting a non-existent user."""
        mock_db.get_user.return_value = None
        
        with pytest.raises(UserNotFoundError):
            get_user(user_id=999)
    
    def test_get_user_invalid_id(self):
        """Test with invalid user ID."""
        with pytest.raises(ValueError):
            get_user(user_id=-1)


class TestCreateUser:
    """Tests for create_user handler."""
    
    @patch('src.api.handlers.db')
    def test_create_user_success(self, mock_db):
        """Test successful user creation."""
        mock_db.create_user.return_value = Mock(id=1)
        
        result = create_user(
            name="John Doe",
            email="john@example.com"
        )
        
        assert result["id"] == 1
        assert result["message"] == "User created successfully"
    
    @patch('src.api.handlers.db')
    def test_create_user_duplicate_email(self, mock_db):
        """Test creating user with existing email."""
        mock_db.create_user.side_effect = DuplicateEmailError()
        
        with pytest.raises(DuplicateEmailError):
            create_user(
                name="John Doe",
                email="existing@example.com"
            )
```

---

### Summary

| File | Tests Generated | Coverage Areas |
|------|-----------------|----------------|
| `src/utils/validator.py` | 15 tests | Email, phone, URL validation |
| `src/api/handlers.py` | 8 tests | CRUD operations, error handling |

**To run these tests:**
```bash
pytest tests/ -v
```

---
*These tests are generated based on the code changes. Please review and adjust 
as needed for your specific requirements.*
```

**Tips:**
- AI sẽ detect testing framework từ project (pytest, unittest, jest, etc.)
- Tests bao gồm happy path, edge cases, và error cases
- Mock external dependencies tự động
- Có thể customize: `@reviewer generate tests with 100% coverage`

---

### `@reviewer tests for <file>`

Yêu cầu AI generate tests cho một file cụ thể.

**Cách dùng:**
```
@reviewer tests for <file_path>
```

**Nơi sử dụng:** Bất kỳ comment nào trong PR

**Ví dụ:**
```
@reviewer tests for src/services/payment.py
```

```
@reviewer tests for src/components/UserProfile.tsx
```

**Response tương tự như `generate tests` nhưng chỉ cho file được chỉ định.**

**Tips:**
- File path có thể relative hoặc absolute
- Có thể chỉ định multiple files: `@reviewer tests for src/a.py and src/b.py`
- Có thể focus vào specific function: `@reviewer tests for src/utils.py focusing on parse_date`

---

## Advanced Usage

### Combining Commands với Context

Bạn có thể thêm context vào commands để AI hiểu rõ hơn yêu cầu:

```
@reviewer fix this, but maintain backward compatibility with v1 API
```

```
@reviewer explain in simple terms for junior developers
```

```
@reviewer generate tests using pytest-asyncio for async functions
```

### Follow-up Questions

Sau khi AI response, bạn có thể tiếp tục hỏi trong cùng thread:

```
User: @reviewer explain
AI: [Detailed explanation about SQL injection...]

User: @reviewer can you show more examples of parameterized queries in SQLAlchemy?
AI: [Additional examples...]
```

### Language Preference

Commands sẽ response bằng ngôn ngữ được cấu hình trong `.reviewer.yaml`:

```yaml
language: "vi"  # Response bằng tiếng Việt
```

Hoặc override trong command:
```
@reviewer explain in English
```

---

## Cấu hình Chat Commands

### Enable/Disable Chat

```yaml
chat:
  enabled: true  # Bật/tắt chat commands
```

### Restrict Commands

Chỉ cho phép một số commands nhất định:

```yaml
chat:
  enabled: true
  allowed_commands:
    - help
    - explain
    - fix
    # - tests  # Disabled
```

### Rate Limiting

Để tránh abuse, hệ thống có rate limiting:

- Max 10 commands per PR per hour
- Max 3 concurrent requests per user
- Cooldown 30 seconds giữa các commands

---

## Troubleshooting

### Command không được nhận

**Nguyên nhân có thể:**
1. Bot chưa được mention đúng cách
2. Chat commands bị disable trong config
3. Command không nằm trong `allowed_commands`

**Giải pháp:**
- Đảm bảo mention đúng: `@reviewer` (không phải `@Reviewer` hoặc `@ reviewer`)
- Kiểm tra `.reviewer.yaml` có `chat.enabled: true`
- Kiểm tra command có trong `allowed_commands`

### Response chậm hoặc timeout

**Nguyên nhân có thể:**
1. PR có quá nhiều files/changes
2. LLM API đang chậm
3. Rate limit

**Giải pháp:**
- Đợi và thử lại sau vài phút
- Sử dụng `tests for <specific_file>` thay vì `generate tests` cho toàn bộ PR

### Fix suggestion không chính xác

**Nguyên nhân có thể:**
1. AI thiếu context về codebase
2. Vấn đề phức tạp cần human judgment

**Giải pháp:**
- Thêm context vào command: `@reviewer fix this considering our custom ORM`
- Review và adjust suggestion trước khi apply
- Sử dụng `@reviewer explain` trước để hiểu vấn đề

---

## Best Practices

### 1. Sử dụng `explain` trước `fix`

Hiểu vấn đề trước khi yêu cầu fix giúp bạn đánh giá suggestion tốt hơn.

### 2. Review generated tests

Tests được generate tự động có thể thiếu edge cases specific cho business logic của bạn. Luôn review và bổ sung.

### 3. Provide context khi cần

AI hoạt động tốt hơn khi có đủ context:
```
@reviewer fix this - we're using SQLAlchemy 2.0 with async sessions
```

### 4. Không rely hoàn toàn vào AI

AI suggestions là điểm khởi đầu, không phải final solution. Human review vẫn cần thiết.
