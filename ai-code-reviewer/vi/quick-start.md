# Bắt đầu nhanh

Hướng dẫn cài đặt AI Code Reviewer cho repository của bạn trong 5 phút.

## Bước 1: Cài đặt GitHub App

1. Truy cập link cài đặt GitHub App (liên hệ admin để lấy link)
2. Chọn repository muốn enable
3. Click **Install**

Vậy là xong! AI Code Reviewer sẽ tự động review mọi PR mới.

## Bước 2: Tùy chỉnh (Tùy chọn)

Tạo file `.reviewer.yaml` ở root repository để tùy chỉnh:

```yaml
# Ngôn ngữ review comments
language: "vi"  # vi, en, ja

# Độ nghiêm ngặt
reviews:
  profile: "default"  # chill, default, strict
```

Xem thêm: [Cấu hình chi tiết](user-guide/configuration.md)

## Bước 3: Tạo Pull Request

Tạo một PR để test:

```bash
git checkout -b test/ai-reviewer
echo "# Test" > test.md
git add . && git commit -m "Test AI Reviewer"
git push -u origin test/ai-reviewer
```

Mở PR trên GitHub và đợi khoảng 1-2 phút. AI sẽ tự động review và comment.

---

## Tiếp theo

- [Tính năng Auto Review](user-guide/auto-review.md)
- [Chat Commands](user-guide/chat-commands.md)
- [Cấu hình chi tiết](user-guide/configuration.md)
