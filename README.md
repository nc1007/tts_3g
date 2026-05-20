Dưới đây là nội dung tài liệu được viết lại chuẩn định dạng Markdown, sửa cú pháp bảng biểu, code block, và loại bỏ các dấu hiệu trích dẫn không cần thiết:

```markdown
# Tài liệu Hướng dẫn Sử dụng API OmniVoice (Voice Cloning Service)

Tài liệu này hướng dẫn chi tiết cách tích hợp và sử dụng hệ thống API tổng hợp giọng nói và nhân bản giọng nói (Voice Cloning) dựa trên mô hình **OmniVoice** và framework **FastAPI**.

---

## 1. Tổng quan hệ thống

Hệ thống cung cấp giải pháp nhân bản giọng nói (Zero-shot Voice Cloning) chất lượng cao bằng cách tiếp nhận một file âm thanh mẫu (Reference Audio) cùng văn bản tương ứng (Reference Text) để tạo ra định danh người nói (`id_directory`), từ đó cho phép tổng hợp một văn bản mới bất kỳ với chất giọng tương tự.

- **Mô hình cốt lõi:** `k2-fsa/OmniVoice` (chạy trên CUDA/CPU, tối ưu FP16)
- **Lưu trữ cấu trúc mẫu:** thư mục cục bộ `voice_sample/`
- **Lưu trữ đám mây:** MinIO S3 (`minio.zoffice.vn`) để lưu và phân phối audio lâu dài
- **Tiền xử lý văn bản:** chuẩn hóa tiếng Việt (`vinorm`), xử lý viết tắt (hđnd → hội đồng nhân dân, ubnd → ủy ban nhân dân), ký tự đặc biệt, ngày tháng, từ ngoại ngữ (AI → Ây Ai)
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

Tải lên file âm thanh mẫu kèm nội dung văn bản tương ứng để tạo định danh duy nhất (`id_directory`). ID sinh ra có tiền tố cố định `vois_`.

- **Phương thức:** `POST`
- **Đường dẫn:** `/create_ID`
- **Content-Type:** `multipart/form-data`

#### 📥 Tham số Request Body

| Tham số | Kiểu dữ liệu | Bắt buộc | Mô tả |
|---------|--------------|----------|-------|
| `audio_file` | File (binary) | Có | File âm thanh mẫu (`.wav`, `.mp3`, `.flac`, …). Hệ thống tự động chuyển đổi sang `.wav` bằng `ffmpeg` nếu cần. |
| `text_voice` | String | Có | Nội dung văn bản chính xác của đoạn âm thanh mẫu. |

#### 📤 Phản hồi (HTTP 200 OK)

```json
{
  "message": "Synthesis complete",
  "directory": "voice_sample/vois_20260520_131005_4821",
  "audio_file": "voice_sample/vois_20260520_131005_4821/sample.wav",
  "text_file": "voice_sample/vois_20260520_131005_4821/transcription.txt",
  "id_directory": "vois_20260520_131005_4821",
  "recognized_text": "Xin chào, đây là đoạn âm thanh mẫu để hệ thống học giọng nói."
}
```

#### 💻 Ví dụ tích hợp

**cURL:**

```bash
curl -X POST \
  'http://localhost:7009/create_ID' \
  -H 'accept: application/json' \
  -F 'audio_file=@giong_mau.wav;type=audio/wav' \
  -F 'text_voice=Xin chào, đây là đoạn âm thanh mẫu để hệ thống học giọng nói.'
```

**Python:**

```python
import requests

url = "http://localhost:7009/create_ID"
payload = {"text_voice": "Xin chào, đây là đoạn âm thanh mẫu để hệ thống học giọng nói."}
files = {"audio_file": ("giong_mau.wav", open("giong_mau.wav", "rb"), "audio/wav")}

response = requests.post(url, data=payload, files=files)
print(response.json())
```

---

### 3.2. Tổng hợp giọng nói / Nhân bản giọng mẫu V2 (`/synthesize_ID_v2`)

Dùng ID giọng nói đã khởi tạo để tổng hợp văn bản mới thành âm thanh.

- **Phương thức:** `POST`
- **Đường dẫn:** `/synthesize_ID_v2`
- **Content-Type:** `multipart/form-data`

#### 📥 Tham số Request Body

| Tham số | Kiểu dữ liệu | Bắt buộc | Mặc định | Mô tả |
|---------|--------------|----------|----------|-------|
| `text` | String | Có | — | Đoạn văn bản cần tổng hợp. |
| `id` | String | Có | — | Định danh thư mục giọng mẫu (VD: `vois_20260520_131005_4821`). |
| `speed` | Float | Không | `1.0` | Tốc độ nói (`0.8` chậm, `1.2` nhanh). |
| `output_type` | String | Không | `binary` | `binary` – trả về file audio trực tiếp; `minio` – tải lên MinIO và trả về URL. |
| `language` | String | Không | `None` | Ngôn ngữ (VD: `vi`, `Vietnamese`). Bỏ trống để tự động nhận diện. |
| `num_step` | Integer | Không | `32` | Số bước lấy mẫu (sampling steps) trong mô hình diffusion. |
| `guidance_scale` | Float | Không | `2.0` | Mức độ bám sát văn bản. |

#### 📤 Phản hồi

**Trường hợp `output_type = binary` (mặc định):**

- HTTP `200 OK`
- `Content-Type: audio/wav`
- Body là luồng binary file âm thanh.

**Trường hợp `output_type = minio`:**

- HTTP `200 OK`
- `Content-Type: application/json`

```json
{
  "message": "Synthesis complete",
  "url": "https://minio.zoffice.vn/zipai/synthesized/vois_20260520_131005_4821/20260520_131140_aBc123XyZ7.wav",
  "output_type": "minio"
}
```

#### 💻 Ví dụ tích hợp

**cURL (lưu binary ra file):**

```bash
curl -X POST \
  'http://localhost:7009/synthesize_ID_v2' \
  -H 'Content-Type: multipart/form-data' \
  -F 'text=Hội đồng nhân dân tỉnh quyết định áp dụng công nghệ AI vào đời sống.' \
  -F 'id=vois_20260520_131005_4821' \
  -F 'output_type=binary' \
  --output ket_qua.wav
```

**Python (lấy URL MinIO):**

```python
import requests

url = "http://localhost:7009/synthesize_ID_v2"
payload = {
    "text": "Hội đồng nhân dân tỉnh quyết định áp dụng công nghệ AI vào đời sống.",
    "id": "vois_20260520_131005_4821",
    "output_type": "minio",
    "speed": 1.0
}

response = requests.post(url, data=payload)
print(response.json())
```

---

### 3.3. Lấy danh sách các ID giọng nói (`/list_ids`)

Liệt kê tất cả định danh giọng nói đã khởi tạo.

- **Phương thức:** `GET`
- **Đường dẫn:** `/list_ids`

#### 📤 Phản hồi (HTTP 200 OK)

```json
{
  "id_directories": [
    "vois_20260520_131005_4821",
    "vois_20260519_091522_1102"
  ]
}
```

#### 💻 Ví dụ tích hợp

**cURL:**

```bash
curl -X GET 'http://localhost:7009/list_ids' -H 'accept: application/json'
```

**Python:**

```python
import requests

response = requests.get("http://localhost:7009/list_ids")
print(response.json())
```

---

### 3.4. Xóa một định danh giọng nói (`/delete_id/{timestamp}`)

Xóa hoàn toàn thư mục dữ liệu mẫu của một ID.

- **Phương thức:** `DELETE`
- **Đường dẫn:** `/delete_id/{timestamp}`
- **Tham số đường dẫn:**
  - `timestamp`: phần sau tiền tố `vois_` (VD: ID `vois_20260520_131005_4821` → timestamp `20260520_131005_4821`)

#### 📤 Phản hồi (HTTP 200 OK)

```json
{
  "message": "Directory with timestamp 20260520_131005_4821 deleted successfully."
}
```

#### 💻 Ví dụ tích hợp

**cURL:**

```bash
curl -X DELETE 'http://localhost:7009/delete_id/20260520_131005_4821' -H 'accept: application/json'
```

**Python:**

```python
import requests

timestamp = "20260520_131005_4821"
response = requests.delete(f"http://localhost:7009/delete_id/{timestamp}")
print(response.json())
```

---

## 4. Quy trình tiền xử lý dữ liệu ngầm (Pipeline nội bộ)

Sau khi nhận yêu cầu, hệ thống tự động thực hiện chuỗi tiền xử lý văn bản để đảm bảo chất lượng tổng hợp:

1. **Chuyển đổi dấu câu (`convert_punctuation`):**  
   Chỉ giữ lại dấu chấm (`.`) và dấu phẩy (`,`). Các dấu `;`, `:`, `-`, `(`, `)` được chuyển thành dấu phẩy; ký tự `&` thay bằng chữ `và`.

2. **Xử lý mốc thời gian:**  
   Tìm các chuỗi định dạng ngày tháng/năm (VD: `2025-2026`, `20/05/2026`) và thêm dấu phẩy bao quanh để mô hình đọc tách biệt.

3. **Hạ chữ thường & chuẩn hóa văn bản (`normalize_text` & `TTSnorm`):**  
   Chuyển toàn bộ sang chữ thường, áp dụng bộ chuẩn hóa tiếng Việt để xử lý viết tắt (hđnd → hội đồng nhân dân) và từ ngoại ngữ (AI → Ây Ai).

4. **Kiểm lỗi chính tả (`correct_text`):**  
   Gửi văn bản đến dịch vụ sửa lỗi nội bộ tại `http://host.docker.internal:8000/correct`.  
   Nếu dịch vụ lỗi hoặc timeout (> 5s), hệ thống tự động dùng văn bản gốc để đảm bảo tính sẵn sàng.
```
