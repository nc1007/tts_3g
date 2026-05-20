Dưới đây là nội dung file `README.md` được viết lại chuẩn định dạng Markdown dựa trên thông tin trích xuất từ ảnh chụp màn hình (đã sửa lỗi chính tả, bổ sung cấu trúc còn thiếu). Những phần không rõ hoặc bị cắt sẽ được giữ nguyên dưới dạng ghi chú.

```markdown
# Tài liệu Hướng dẫn Sử dụng API OmniVoice (Voice Cloning Service)

Tài liệu này hướng dẫn chi tiết cách tích hợp và sử dụng hệ thống API tổng hợp giọng nói và nhân bản giọng nói (Voice Cloning).

---

## 1. Tổng quan hệ thống

Hệ thống cung cấp giải pháp nhân bản giọng nói (Zero-shot Voice Cloning) chất lượng cao với các thành phần chính:

- **Mô hình cốt lõi:** `k2-fsa/OmniVoice` (chạy trên CUDA/CPU, tối ưu FP16)
- **Lưu trữ cấu trúc mẫu:** thư mục cục bộ `voice_sample`
- **Lưu trữ đám mây:** MinIO S3 (`minio.zoffice.vn`) để lưu và phân phối audio lâu dài
- **Tiền xử lý văn bản:** chuẩn hóa tiếng Việt (`vinorm`), xử lý viết tắt (hđnd -> hội đồng nhân dân, ...)
- **Dịch vụ kiểm lỗi chính tả:** kết nối đồng bộ qua endpoint `http://host.docker.internal:8000/correct`

---

## 2. Thông tin cấu hình Base URL

Mặc định hệ thống khởi chạy với cấu hình:

- **Host:** `0.0.0.0`
- **Port:** `7009`
- **Base URL:** `http://<your-server-ip>:7009`

---

## 3. Danh sách Endpoints chi tiết

### 3.1. Đăng ký & khởi tạo giọng mẫu (`/create_ID`)

Tải lên file âm thanh mẫu kèm nội dung văn bản tương ứng để tạo định danh duy nhất (`id_directory`).

- **Phương thức:** `POST`
- **Đường dẫn:** `/create_ID`
- **Content-Type:** `multipart/form-data`

#### 📥 Tham số Request Body

| Tham số | Kiểu dữ liệu | Bắt buộc | Mô tả |
|---------|--------------|----------|-------|
| `audio_file` | File (binary) | Có | File âm thanh mẫu (`.wav`, `.mp3`, `.flac`, …). Hệ thống tự động chuyển đổi nếu cần. |
| `text_voice` | String | Có | Nội dung văn bản chính xác của đoạn âm thanh mẫu. |

#### 📤 Phản hồi mẫu (HTTP 200 OK)

```json
{
  "message": "Synthesis complete",
  "status": 200,
  "data": {
    "audio_file": "path/to/sample.wav",
    "text_voice": "Nội dung văn bản mẫu",
    "id_directory": "vois_YYYYMMDD_HHMMSS_xxxx"
  }
}
```

> **Lưu ý:** Các endpoint còn lại (`/synthesize_ID_v2`, `/list_ids`, `/delete_id`) không được hiển thị đầy đủ trong ảnh chụp. Vui lòng tham khảo tài liệu gốc hoặc liên hệ quản trị viên để có thông tin chi tiết.

---

## Giấy phép & Đóng góp

- **Releases:** Chưa có bản phát hành chính thức.
- **Packages:** Chưa có package nào được publish.
- **Contributors:** [nc1007](https://github.com/nc1007)

---

*Tài liệu được tái tạo từ nội dung ảnh chụp màn hình ngày 20/05/2026. Một số phần có thể bị thiếu hoặc không chính xác do nguồn dữ liệu không đầy đủ.*
```

Nếu bạn có file Markdown hoàn chỉnh (như đã cung cấp ở câu hỏi trước), hãy dùng file đó thay vì bản tái tạo từ ảnh chụp. Bạn có muốn tôi chuyển đổi lại theo đúng nội dung gốc trước đó không?
