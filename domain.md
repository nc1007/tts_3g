# Tài liệu Hướng dẫn Sử dụng API Voice

Tài liệu này hướng dẫn chi tiết cách tích hợp và sử dụng hệ thống API tổng hợp giọng nói và nhân bản giọng nói (Voice Cloning) dựa trên mô hình **Voice** và framework **FastAPI**.

---

## 1. Tổng quan hệ thống

Hệ thống cung cấp giải pháp nhân bản giọng nói (Zero-shot Voice Cloning) chất lượng cao bằng cách tiếp nhận một file âm thanh mẫu (Reference Audio) cùng văn bản tương ứng (Reference Text) để tạo ra định danh người nói (`id_directory`), từ đó cho phép tổng hợp một văn bản mới bất kỳ với chất giọng tương tự.

| Thành phần | Mô tả |
| :--- | :--- |
| **Mô hình cốt lõi** | `Voice` — chạy trên môi trường CUDA/CPU, tối ưu hóa FP16 |
| **Lưu trữ cấu trúc mẫu** | Lưu cục bộ tại thư mục `voice_sample/` |
| **Lưu trữ đám mây** | Tích hợp **MinIO** S3 (`minio.zoffice.vn`) để lưu trữ và phân phối file audio |
| **Tiền xử lý văn bản** | Bộ chuẩn hóa tiếng Việt `vinorm` — xử lý từ viết tắt, ký tự đặc biệt, ngày tháng, ngoại ngữ (ví dụ: *AI* → *Ây Ai*) |
| **Kiểm lỗi chính tả** | Dịch vụ sửa lỗi ngoài tại `http://host.docker.internal:8000/correct` |

---

## 2. Cấu hình Base URL

**Base URL:** `https://api.voice.vn`

---

## 3. Danh sách Endpoints

### 3.1. Đăng ký & Khởi tạo giọng mẫu — `POST /create_ID`

Tải lên file âm thanh mẫu kèm nội dung văn bản tương ứng để tạo ra một định danh duy nhất (`id_directory`). ID sinh ra có tiền tố cố định là `vois_`.

- **Content-Type:** `multipart/form-data`

#### Tham số Request Body

| Tham số | Kiểu | Bắt buộc | Mô tả |
| :--- | :--- | :---: | :--- |
| `audio_file` | File (Binary) | ✅ | File âm thanh mẫu (`.wav`, `.mp3`, `.flac`,...). Nếu không phải `.wav`, hệ thống tự động chuyển đổi qua `ffmpeg`. |
| `text_voice` | String | ✅ | Nội dung văn bản chính xác của đoạn âm thanh mẫu, dùng làm dữ liệu đối chiếu cho mô hình. |

#### Phản hồi mẫu — `200 OK`

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

#### Ví dụ tích hợp

**cURL:**

```bash
curl -X POST 'https://api.voice.vn/create_ID' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'audio_file=@giong_mau.wav;type=audio/wav' \
  -F 'text_voice=Xin chào, đây là đoạn âm thanh mẫu để hệ thống học giọng nói.'
```

**Python:**

```python
import requests

url = "https://api.voice.vn/create_ID"
payload = {"text_voice": "Xin chào, đây là đoạn âm thanh mẫu để hệ thống học giọng nói."}
files = {"audio_file": ("giong_mau.wav", open("giong_mau.wav", "rb"), "audio/wav")}

response = requests.post(url, data=payload, files=files)
print(response.json())
```

---

### 3.2. Tổng hợp giọng nói — `POST /synthesize_ID_v2`

Sử dụng ID giọng nói đã khởi tạo để tổng hợp một văn bản mới thành âm thanh với chất giọng tương tự bản gốc.

- **Content-Type:** `multipart/form-data`

#### Tham số Request Body

| Tham số | Kiểu | Bắt buộc | Mặc định | Mô tả |
| :--- | :--- | :---: | :--- | :--- |
| `text` | String | ✅ | — | Đoạn văn bản cần tổng hợp thành tiếng nói. |
| `id` | String | ✅ | — | ID thư mục giọng mẫu (ví dụ: `vois_20260520_131005_4821`). |
| `speed` | Float | — | `1.0` | Tốc độ nói. `0.8` = chậm hơn, `1.2` = nhanh hơn. |
| `output_type` | String | — | `binary` | Hình thức trả kết quả: `binary` (stream audio) hoặc `minio` (trả URL cloud). |
| `language` | String | — | `None` | Ngôn ngữ đầu vào (ví dụ: `vi`, `Vietnamese`). Để trống để tự nhận diện. |
| `num_step` | Integer | — | `32` | Số bước lấy mẫu (Sampling Steps) của mô hình Diffusion. |
| `guidance_scale` | Float | — | `2.0` | Thang đo hướng dẫn — kiểm soát độ bám sát văn bản đầu vào. |

#### Phản hồi mẫu

**Khi `output_type = binary` (mặc định):**

- **HTTP:** `200 OK`
- **Content-Type:** `audio/wav`
- Trả về stream binary trực tiếp. Client có thể lưu file hoặc gắn vào thẻ `<audio>` để phát.

**Khi `output_type = minio`:**

- **HTTP:** `200 OK`
- **Content-Type:** `application/json`

```json
{
  "message": "Synthesis complete",
  "url": "https://minio.zoffice.vn/zipai/synthesized/vois_20260520_131005_4821/20260520_131140_aBc123XyZ7.wav",
  "output_type": "minio"
}
```

#### Ví dụ tích hợp

**cURL (nhận stream binary về file):**

```bash
curl -X POST 'https://api.voice.vn/synthesize_ID_v2' \
  -H 'Content-Type: multipart/form-data' \
  -F 'text=Hội đồng nhân dân tỉnh quyết định áp dụng công nghệ AI vào đời sống.' \
  -F 'id=vois_20260520_131005_4821' \
  -F 'output_type=binary' \
  --output ket_qua.wav
```

**Python (nhận URL MinIO):**

```python
import requests

url = "https://api.voice.vn/synthesize_ID_v2"
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

### 3.3. Lấy danh sách ID giọng nói — `GET /list_ids`

Liệt kê tất cả các `id_directory` đã được khởi tạo và đang lưu trữ trên hệ thống.

#### Phản hồi mẫu — `200 OK`

```json
{
  "id_directories": [
    "vois_20260520_131005_4821",
    "vois_20260519_091522_1102"
  ]
}
```

#### Ví dụ tích hợp

**cURL:**

```bash
curl -X GET 'https://api.voice.vn/list_ids' \
  -H 'accept: application/json'
```

**Python:**

```python
import requests

response = requests.get("https://api.voice.vn/list_ids")
print(response.json())
```

---

### 3.4. Xóa một định danh giọng nói — `DELETE /delete_id/{timestamp}`

Xóa toàn bộ thư mục dữ liệu mẫu của một ID giọng nói để giải phóng dung lượng.

- **Tham số đường dẫn:** `timestamp` — phần ký tự đứng sau tiền tố `vois_`.

> **Ví dụ:** Với ID `vois_20260520_131005_4821`, giá trị `timestamp` cần truyền là `20260520_131005_4821`.

#### Phản hồi mẫu — `200 OK`

```json
{
  "message": "Directory with timestamp 20260520_131005_4821 deleted successfully."
}
```

#### Ví dụ tích hợp

**cURL:**

```bash
curl -X DELETE 'https://api.voice.vn/delete_id/20260520_131005_4821' \
  -H 'accept: application/json'
```

**Python:**

```python
import requests

timestamp = "20260520_131005_4821"
response = requests.delete(f"https://api.voice.vn/delete_id/{timestamp}")
print(response.json())
```

---

## 4. Quy trình tiền xử lý văn bản (Pipeline nội bộ)

Để đảm bảo chất lượng giọng đọc tự nhiên và tránh lỗi mô hình, hệ thống tự động chạy chuỗi xử lý sau trước khi tổng hợp:

1. **Chuyển đổi dấu câu (`convert_punctuation`)**
   Chỉ giữ lại dấu chấm (`.`) và dấu phẩy (`,`). Các ký tự `;`, `:`, `-`, `(`, `)` được chuyển thành dấu phẩy để tạo ngắt nghỉ tự nhiên. Ký tự `&` được thay bằng chữ `và`.

2. **Xử lý mốc thời gian**
   Tự động nhận diện các chuỗi ngày tháng/năm (ví dụ: `2025-2026`, `20/05/2026`) và thêm dấu phẩy bao quanh để mô hình đọc tách biệt, mạch lạc.

3. **Hạ chữ thường & Chuẩn hóa văn bản (`normalize_text` & `TTSnorm`)**
   Chuyển toàn bộ chuỗi sang chữ thường, sau đó chuẩn hóa qua thư viện tiếng Việt chuyên dụng: giải nghĩa từ viết tắt hành chính và phát âm thuật ngữ tiếng Anh (ví dụ: *hđnd* → *hội đồng nhân dân*, *AI* → *Ây Ai*).

4. **Kiểm lỗi chính tả (`correct_text`)**
   Gửi văn bản qua dịch vụ Spelling Correction nội bộ để chỉnh lỗi gõ trước khi đưa vào mô hình. Nếu dịch vụ lỗi hoặc timeout (> 5 giây), hệ thống tự động dùng văn bản gốc để đảm bảo tính sẵn sàng cao.
