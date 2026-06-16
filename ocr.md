# OCR + Invoice Parser API + phân tích hình ảnh — Tài liệu API

Tài liệu mô tả các endpoint của service FastAPI dùng để OCR ảnh/PDF, parse hóa đơn xăng dầu, và mô tả ảnh bằng VLM.

**Base URL:** `https://ocr-hd.zipai.vn`
**Version:** 2.0.0

---

## Mục lục

- [1. POST /ocr — OCR (ảnh được chuyển sang PDF trước khi OCR)](#1-post-ocr)
- [2. POST /ocr_base — Giống /ocr (alias)](#2-post-ocr_base)
- [3. POST /invoice — OCR + Parse hóa đơn xăng dầu](#3-post-invoice)
- [4. POST /describe — Mô tả ảnh bằng VLM ](#4-post-describe)
- [5. GET /health — Health check](#5-get-health)
- [Phụ lục: Định dạng file hỗ trợ](#phụ-lục-định-dạng-file-hỗ-trợ)

---

## 1. POST `/ocr`

OCR ảnh hoặc PDF. Nếu input là ảnh, ảnh sẽ được chuyển đổi sang PDF (xử lý EXIF orientation, convert RGB) và lưu vào thư mục `pdf/` trước khi OCR. Text kết quả từ ảnh cũng được chạy qua `extract_cccd`.

### Request

`multipart/form-data`

| Field  | Type | Required | Mô tả                              |
|--------|------|----------|-------------------------------------|
| `file` | File | ✅        | Ảnh (jpg/png/webp/gif/bmp) hoặc PDF |

### Response `200 OK`

**Trường hợp input là ảnh:**

```json
{
  "filename": "anh.jpg",
  "pdf_saved": "pdf/anh.pdf",
  "pages": 1,
  "elapsed": 2.05,
  "text": "...",
  "pages_text": ["..."]
}
```

**Trường hợp input là PDF:**

```json
{
  "filename": "hoadon.pdf",
  "pages": 2,
  "elapsed": 3.10,
  "text": "=== TRANG 1 ===\n...\n\n=== TRANG 2 ===\n...",
  "pages_text": ["...", "..."]
}
```

| Field        | Type     | Mô tả                                                |
|--------------|----------|--------------------------------------------------------|
| `filename`   | string   | Tên file gốc                                            |
| `pdf_saved`  | string   | (chỉ có khi input là ảnh) Đường dẫn PDF đã được tạo và lưu |
| `pages`      | int      | Số trang                                                |
| `elapsed`    | float    | Thời gian xử lý (giây)                                  |
| `text`       | string   | Toàn bộ text OCR                                        |
| `pages_text` | string[] | Text OCR theo từng trang                                 |

### Lỗi

| Code | Khi nào                                  |
|------|---------------------------------------------|
| 400  | Không đọc được PDF                          |
| 415  | Định dạng file không hỗ trợ                 |
| 500  | Lỗi chuyển ảnh sang PDF hoặc lỗi OCR PDF     |

### Ví dụ curl

```bash
curl -X POST "https://ocr-hd.zipai.vn/ocr" \
  -F "file=@/path/to/anh.jpg"
```

---

## 2. POST `/ocr_base`

Hành vi **giống hoàn toàn** endpoint `/ocr` (ảnh → chuyển sang PDF → OCR; ảnh có chạy `extract_cccd`). Request/response identical với mục [1. POST /ocr](#1-post-ocr).


### Ví dụ curl

```bash
curl -X POST "https://ocr-hd.zipai.vn/ocr_base" \
  -F "file=@/path/to/anh.png"
```

---

## 3. POST `/invoice`

OCR ảnh/PDF hóa đơn xăng dầu, sau đó parse thành JSON có cấu trúc (số hóa đơn, ngày, loại nhiên liệu, số lượng, đơn giá, tổng tiền, v.v. — tùy định nghĩa trong `invoice_parser.py`).

- Nếu input là **ảnh**: chuyển sang PDF, đồng thời gọi `extract_invoice_info(pdf_path)` (từ `vllm_ocr.py`) để lấy thêm thông tin bổ sung, rồi OCR PDF và nối kết quả `extract_invoice_info` vào cuối text OCR trước khi parse.
- Nếu input là **PDF**: OCR trực tiếp.
- Kết quả OCR được đưa qua `parse_invoice` (LLM-based) hoặc `parse_invoice_regex` (regex-based) tùy tham số `use_llm_parser`.

### Request

`multipart/form-data` + query param

| Field            | Type    | Required | Default | Mô tả                                            |
|------------------|---------|----------|---------|-----------------------------------------------------|
| `file`           | File    | ✅        | —       | Ảnh hoặc PDF hóa đơn                                |
| `use_llm_parser` | bool (query) | ❌   | `true`  | `true`: parse bằng LLM; `false`: parse bằng regex   |

### Response `200 OK`

Cấu trúc JSON phụ thuộc vào implementation của `parse_invoice` / `parse_invoice_regex` (định nghĩa trong `invoice_parser.py`). Thường bao gồm các trường như tên hóa đơn, ngày, mặt hàng, số lượng, đơn giá, thành tiền, `elapsed`, `filename`...

### Lỗi

| Code | Khi nào                                       |
|------|--------------------------------------------------|
| 400  | Không đọc được PDF                                |
| 415  | Định dạng file không hỗ trợ                       |
| 500  | Lỗi chuyển ảnh sang PDF hoặc lỗi OCR PDF           |

### Ví dụ curl

```bash
# Dùng LLM parser (mặc định)
curl -X POST "https://ocr-hd.zipai.vn/invoice" \
  -F "file=@/path/to/hoadon.jpg"

# Dùng regex parser
curl -X POST "https://ocr-hd.zipai.vn/invoice?use_llm_parser=false" \
  -F "file=@/path/to/hoadon.pdf"
```

---

## 4. POST `/describe`

Upload 1 ảnh kèm câu prompt text, gửi tới model VLM (qua endpoint `client1` tại `http://localhost:1234/v1`) để mô tả/trả lời theo prompt.

### Request

`multipart/form-data`

| Field   | Type   | Required | Default                            | Mô tả                  |
|---------|--------|----------|-------------------------------------|--------------------------|
| `image` | File   | ✅        | —                                   | File ảnh (bất kỳ mime image) |
| `text`  | string | ❌        | `"Describe this image in detail."` | Câu hỏi/prompt cho ảnh   |

### Response `200 OK`

```json
{
  "result": "Mô tả hoặc câu trả lời của model về ảnh..."
}
```

### Ví dụ curl

```bash
curl -X POST "https://ocr-hd.zipai.vn/describe" \
  -F "image=@/path/to/cat.jpg" \
  -F "text=Mô tả chi tiết bức ảnh này."
```

---

## 5. GET `/health`

Health check đơn giản.

### Response `200 OK`

```json
{ "status": "ok" }
```

### Ví dụ curl

```bash
curl https://ocr-hd.zipai.vn/health
```

---

## Phụ lục: Định dạng file hỗ trợ

| Loại  | Đuôi file                                  |
|-------|----------------------------------------------|
| Ảnh   | `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.bmp` |
| PDF   | `.pdf`                                        |

Gửi file có đuôi khác sẽ trả về lỗi `415 Unsupported Media Type`.


> **Lưu ý:** `/ocr`, `/ocr_base` đều cùng dùng model OCR ở trên; `/describe` dùng model VLM riêng trên cổng `1234`.
