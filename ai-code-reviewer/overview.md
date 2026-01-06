# AI Code Reviewer

AI Code Reviewer là hệ thống review code tự động sử dụng AI, được thiết kế để phân tích Pull Requests trên GitHub thông qua nhiều AI agents chuyên biệt. Hệ thống được xây dựng trên nền tảng LangGraph, cho phép các agents hoạt động song song và phối hợp với nhau để đưa ra những đánh giá toàn diện về code.

## Tại sao cần AI Code Reviewer?

Code review là một phần quan trọng trong quy trình phát triển phần mềm, nhưng thường gặp những thách thức sau:

- **Tốn thời gian**: Reviewer phải đọc và hiểu từng dòng code thay đổi
- **Không nhất quán**: Mỗi reviewer có tiêu chuẩn và kinh nghiệm khác nhau
- **Dễ bỏ sót**: Con người có thể bỏ qua các lỗi nhỏ hoặc vấn đề bảo mật tiềm ẩn
- **Bottleneck**: Khi team có nhiều PR, reviewer trở thành điểm nghẽn

AI Code Reviewer giải quyết những vấn đề này bằng cách:

- **Tự động hóa**: Review ngay khi PR được tạo hoặc cập nhật
- **Nhất quán**: Áp dụng cùng tiêu chuẩn cho mọi PR
- **Toàn diện**: Kiểm tra security, logic, và style cùng lúc
- **Nhanh chóng**: Kết quả review trong vài phút

---

## Kiến trúc hệ thống

Khi một Pull Request được tạo hoặc cập nhật, hệ thống sẽ tự động nhận webhook từ GitHub và bắt đầu quy trình review.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                            │
│                                                                      │
│   Developer tạo/cập nhật Pull Request                               │
│                              │                                       │
│                              ▼                                       │
│                    Webhook Event Triggered                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FastAPI Webhook Handler                         │
│                                                                      │
│   1. Xác thực HMAC-SHA256 signature                                 │
│   2. Parse event payload (PR opened/synchronized/reopened)          │
│   3. Kiểm tra điều kiện auto-review (skip keywords, authors, etc.)  │
│   4. Đẩy task vào Celery queue                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Celery Worker + Redis Queue                       │
│                                                                      │
│   - Xử lý task bất đồng bộ                                          │
│   - Retry tự động khi lỗi (tối đa 3 lần)                            │
│   - Timeout 5 phút cho mỗi task                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LangGraph Review Pipeline                         │
│                                                                      │
│   ┌─────────────┐                                                   │
│   │ Acknowledge │ ──► Post comment "Review started..."              │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │   Extract   │ ──► Fetch PR files, parse diffs                   │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────────────────────────────────────────┐               │
│   │              Parallel Agent Execution            │               │
│   │                                                  │               │
│   │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐        │
│   │  │   Security   │  │    Logic     │  │     Style      │        │
│   │  │    Agent     │  │    Agent     │  │     Agent      │        │
│   │  └──────────────┘  └──────────────┘  └────────────────┘        │
│   │         │                 │                  │                  │
│   │         └─────────────────┼──────────────────┘                  │
│   │                           │                                     │
│   └───────────────────────────┼─────────────────────────────────────┘
│                               │                                      │
│                               ▼                                      │
│   ┌─────────────┐                                                   │
│   │  Aggregate  │ ──► Deduplicate, sort by severity & confidence    │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │   Publish   │ ──► Post review comments to GitHub PR             │
│   └──────┬──────┘                                                   │
│          │                                                          │
│          ▼                                                          │
│   ┌─────────────┐                                                   │
│   │   Notify    │ ──► Send Slack notification (optional)            │
│   └─────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Agent Review System

Hệ thống sử dụng ba AI agents chuyên biệt, mỗi agent tập trung vào một khía cạnh khác nhau của code review. Các agents chạy song song để tối ưu thời gian xử lý.

### Security Agent

Security Agent chịu trách nhiệm phát hiện các lỗ hổng bảo mật trong code. Agent này được train để nhận diện các pattern nguy hiểm phổ biến.

**Các vấn đề được phát hiện:**

| Loại lỗ hổng | Mô tả | Ví dụ |
|--------------|-------|-------|
| **SQL Injection** | Query SQL được xây dựng từ user input không được sanitize | `query = f"SELECT * FROM users WHERE id = {user_id}"` |
| **XSS (Cross-Site Scripting)** | Render user input trực tiếp vào HTML | `innerHTML = userComment` |
| **Hardcoded Secrets** | API keys, passwords, tokens trong source code | `api_key = "sk-1234567890abcdef"` |
| **SSRF (Server-Side Request Forgery)** | Request đến URL do user cung cấp | `requests.get(user_provided_url)` |
| **Command Injection** | Thực thi shell command với user input | `os.system(f"ping {hostname}")` |
| **Path Traversal** | Truy cập file ngoài thư mục cho phép | `open(f"uploads/{filename}")` |
| **Insecure Deserialization** | Deserialize data không tin cậy | `pickle.loads(user_data)` |
| **Weak Cryptography** | Sử dụng thuật toán mã hóa yếu | `hashlib.md5(password)` |

### Logic Agent

Logic Agent tập trung vào việc phát hiện các lỗi logic và edge cases có thể gây ra bugs trong runtime.

**Các vấn đề được phát hiện:**

| Loại lỗi | Mô tả | Ví dụ |
|----------|-------|-------|
| **Null/None Reference** | Truy cập property của object có thể null | `user.name` khi `user` có thể là `None` |
| **Off-by-one Errors** | Lỗi index trong vòng lặp hoặc array | `for i in range(len(arr) + 1)` |
| **Missing Error Handling** | Không xử lý exception có thể xảy ra | `file = open(path)` không có try-except |
| **Race Conditions** | Truy cập shared resource không đồng bộ | Concurrent write không có lock |
| **Infinite Loops** | Vòng lặp không có điều kiện thoát hợp lệ | `while True:` không có `break` |
| **Resource Leaks** | Không đóng file, connection sau khi sử dụng | `open()` không có `close()` hoặc context manager |
| **Type Mismatches** | Sử dụng sai kiểu dữ liệu | String concatenation với integer |
| **Edge Cases** | Không xử lý các trường hợp đặc biệt | Empty array, negative numbers, unicode |

### Style Agent

Style Agent kiểm tra code style, conventions, và maintainability của code.

**Các vấn đề được phát hiện:**

| Loại vấn đề | Mô tả | Ví dụ |
|-------------|-------|-------|
| **Naming Conventions** | Tên biến, hàm không theo convention | `def DoSomething()` thay vì `def do_something()` |
| **Code Formatting** | Indentation, spacing không nhất quán | Mixed tabs và spaces |
| **Dead Code** | Code không được sử dụng | Unused imports, unreachable code |
| **Missing Documentation** | Thiếu docstring cho public functions | Function phức tạp không có comment |
| **Code Duplication** | Logic lặp lại nhiều nơi | Copy-paste code blocks |
| **Complex Functions** | Function quá dài hoặc phức tạp | Cyclomatic complexity cao |
| **Magic Numbers** | Hardcoded numbers không có ý nghĩa | `if status == 3:` |
| **Inconsistent Style** | Style không nhất quán trong cùng file | Mix single và double quotes |

---

## Severity Levels

Mỗi finding từ agents được gán một mức độ nghiêm trọng (severity) để giúp developer ưu tiên xử lý.

| Severity | Màu | Mô tả | Hành động |
|----------|-----|-------|-----------|
| `critical` | Đỏ | Lỗ hổng bảo mật nghiêm trọng, có thể gây mất dữ liệu hoặc bị tấn công | **Bắt buộc** sửa trước khi merge |
| `warning` | Vàng | Bugs, lỗi logic có thể gây crash hoặc behavior sai | **Nên** sửa trước khi merge |
| `info` | Xanh dương | Vấn đề về code quality, best practices | **Cân nhắc** sửa |
| `suggestion` | Xám | Gợi ý cải thiện style, optimization | **Tùy chọn** |

---

## Confidence Scores

Mỗi finding cũng có một confidence score (0.0 - 1.0) thể hiện độ tin cậy của AI về vấn đề được phát hiện.

- **0.9 - 1.0**: Rất chắc chắn, pattern rõ ràng
- **0.8 - 0.9**: Khá chắc chắn, có thể cần verify
- **0.7 - 0.8**: Có khả năng, nên kiểm tra kỹ
- **< 0.7**: Không chắc chắn, mặc định không hiển thị

Mặc định, chỉ những findings có confidence >= 0.7 mới được post lên PR. Ngưỡng này có thể điều chỉnh trong configuration.

---

## Các tính năng chính

### 1. Automatic PR Review

Tự động review khi PR được tạo hoặc cập nhật. Xem chi tiết tại [Automatic Review](auto-review.md).

### 2. Interactive Chat Commands

Tương tác với reviewer qua comments bằng cách mention `@reviewer`. Xem chi tiết tại [Chat Commands](chat-commands.md).

### 3. Slack Notifications

Nhận thông báo qua Slack khi review hoàn thành. Xem chi tiết tại [Slack Notifications](slack-notifications.md).

### 4. Per-Repository Configuration

Tùy chỉnh behavior cho từng repository qua file `.reviewer.yaml`. Xem chi tiết tại [Configuration](configuration.md).

### 5. Multi-language Support

Hỗ trợ review comments bằng nhiều ngôn ngữ:
- English (`en`)
- Tiếng Việt (`vi`)
- 日本語 (`ja`)

---

## Supported Languages & Frameworks

AI Code Reviewer có thể review code của hầu hết các ngôn ngữ lập trình phổ biến:

| Ngôn ngữ | Frameworks/Libraries được hỗ trợ tốt |
|----------|--------------------------------------|
| Python | Django, FastAPI, Flask, SQLAlchemy |
| JavaScript/TypeScript | React, Vue, Angular, Node.js, Express |
| Java | Spring Boot, Hibernate |
| Go | Gin, Echo, standard library |
| Ruby | Rails, Sinatra |
| PHP | Laravel, Symfony |
| C# | .NET Core, ASP.NET |
| Rust | Actix, Rocket |

{% hint style="info" %}
Hệ thống sử dụng LLM nên có thể hiểu và review code của bất kỳ ngôn ngữ nào, nhưng các ngôn ngữ phổ biến sẽ có kết quả chính xác hơn.
{% endhint %}
