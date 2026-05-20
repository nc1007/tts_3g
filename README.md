Dưới đây là toàn bộ nội dung file tài liệu hướng dẫn sử dụng API hoàn chỉnh dưới dạng Markdown (`.md`), bao gồm tổng quan, cấu hình, chi tiết toàn bộ các endpoint (đã bổ sung đầy đủ tham số, ví dụ cURL, ví dụ Python) cùng quy trình tiền xử lý dữ liệu:

```markdown
# Tài liệu Hướng dẫn Sử dụng API Voice (Voice Cloning Service)

Tài liệu này hướng dẫn chi tiết cách tích hợp và sử dụng hệ thống API tổng hợp giọng nói và nhân bản giọng nói (Voice Cloning) dựa trên mô hình **Voice** và framework **FastAPI**.

---

## 1. Tổng quan hệ thống
Hệ thống cung cấp giải pháp nhân bản giọng nói (Zero-shot Voice Cloning) chất lượng cao bằng cách tiếp nhận một file âm thanh mẫu (Reference Audio) cùng văn bản tương ứng (Reference Text) để tạo ra định danh người nói (`id_directory`), từ đó cho phép tổng hợp một văn bản mới bất kỳ với chất giọng tương tự[cite: 1].

* **Mô hình cốt lõi:** `Voice` (Chạy trên môi trường CUDA/CPU, tối ưu hóa FP16)[cite: 1].
* **Hệ thống lưu trữ cấu trúc mẫu:** Lưu cục bộ tại thư mục `voice_sample/`[cite: 1].
* **Hệ thống lưu trữ đám mây:** Tích hợp **MinIO** S3 (`minio.zoffice.vn`) để lưu trữ và phân phối file audio kết quả lâu dài[cite: 1].
* **Tiền xử lý văn bản:** Tích hợp bộ chuẩn hóa tiếng Việt nâng cao (`vinorm`, chuyển đổi từ viết tắt như *hđnd*, *ubnd*, xử lý các ký tự đặc biệt, định dạng ngày tháng, ngoại ngữ như *AI* -> *Ây Ai*)[cite: 1].
* **Dịch vụ kiểm lỗi chính tả:** Kết nối đồng bộ qua dịch vụ sửa lỗi bên ngoài tại đường dẫn `http://host.docker.internal:8000/correct`[cite: 1].

---

## 2. Thông tin cấu hình Base URL
Mặc định hệ thống khởi chạy trên cấu hình cổng mạng sau[cite: 1]:
* **Host:** `0.0.0.0`[cite: 1]
* **Port:** `7009`[cite: 1]
* **Cơ sở URL:** `http://<your-server-ip>:7009`

---

## 3. Danh sách Endpoints chi tiết

### 3.1. Đăng ký & Khởi tạo giọng mẫu (`/create_ID`)
Endpoint này dùng để tải lên file âm thanh mẫu kèm theo nội dung văn bản mà đoạn âm thanh đó đang nói nhằm tạo ra một định danh duy nhất (`id_directory`)[cite: 1]. Cấu trúc ID sinh ra sẽ có tiền tố cố định là `vois_`[cite: 1].

* **Phương thức:** `POST`[cite: 1]
* **Đường dẫn:** `/create_ID`[cite: 1]
* **Kiểu dữ liệu gửi lên (Content-Type):** `multipart/form-data`[cite: 1]

#### 📥 Các tham số Request Body:

| Tham số | Kiểu dữ liệu | Bắt buộc | Mô tả |
| :--- | :--- | :--- | :--- |
| `audio_file` | File (Binary) | **Có** | File âm thanh mẫu chứa giọng nói (Hỗ trợ `.wav`, `.mp3`, `.flac`,... Nếu không phải định dạng `.wav`, hệ thống sẽ tự động dùng `ffmpeg` để chuyển đổi ngầm)[cite: 1]. |
| `text_voice` | String | **Có** | Nội dung văn bản chính xác của đoạn âm thanh mẫu (`audio_file`) đang nói để làm dữ liệu đối chiếu cho mô hình[cite: 1]. |

#### 📤 Phản hồi mẫu (Response - JSON):
* **Mã trạng thái HTTP:** `200 OK`[cite: 1]

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

#### 💻 Ví dụ Code tích hợp:

**cURL:**

```bash
curl -X 'POST' \
  'http://localhost:7009/create_ID' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
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

Sử dụng ID giọng nói đã khởi tạo thành công trước đó để tiến hành tổng hợp một văn bản mới hoàn toàn sang dạng âm thanh (Nhân bản giọng nói dựa trên giọng nền).

* **Phương thức:** `POST`

* **Đường dẫn:** `/synthesize_ID_v2`

* **Kiểu dữ liệu gửi lên (Content-Type):** `multipart/form-data`


#### 📥 Các tham số Request Body:

| Tham số | Kiểu dữ liệu | Bắt buộc | Giá trị mặc định | Mô tả |
| --- | --- | --- | --- | --- |
| `text` | String | **Có** |  | Đoạn văn bản mới cần tổng hợp ra thành tiếng nói.

 |
| `id` | String | **Có** |  | Định danh ID thư mục giọng mẫu (Ví dụ: `vois_20260520_131005_4821`).

 |
| `speed` | Float | Không | `1.0` | Tốc độ nói của âm thanh đầu ra (Ví dụ: `0.8` là nói chậm, `1.2` là nói nhanh hơn).

 |
| `output_type` | String | Không | `binary` | Hình thức trả về kết quả. Nhận hai giá trị lựa chọn:<br>

<br>• `binary`: Trả về trực tiếp luồng dữ liệu file audio (Streaming Response).<br>

<br>• `minio`: Lưu file lên cloud lưu trữ và trả về URL kết nối.

 |
| `language` | String | Không | `None` | Ngôn ngữ xử lý văn bản đầu vào (Ví dụ: `vi`, `Vietnamese`). Nếu bỏ trống (`None`), mô hình sẽ tự động nhận diện ngôn ngữ.

 |
| `num_step` | Integer | Không | `32` | Số bước lấy mẫu (Sampling Steps) để sinh âm thanh qua mô hình tạo mẫu Diffusion.

 |
| `guidance_scale` | Float | Không | `2.0` | Thang đo hướng dẫn điều khiển độ chính xác bám sát văn bản đầu vào của mô hình.

 |

#### 📤 Phản hồi mẫu (Response):

* **Trường hợp `output_type` = `binary` (Mặc định):**
* **Mã trạng thái HTTP:** `200 OK`

* **Content-Type:** `audio/wav`

* **Mô tả:** Trả về file âm thanh dạng Stream Binary trực tiếp. Phía Client có thể tải về trực tiếp hoặc gắn thẳng vào thẻ `<audio>` trên giao diện để phát.




* **Trường hợp `output_type` = `minio`:**
* **Mã trạng thái HTTP:** `200 OK`

* **Content-Type:** `application/json`

* **Nội dung JSON:**


```json
{
  "message": "Synthesis complete",
  "url": "[https://minio.zoffice.vn/zipai/synthesized/vois_20260520_131005_4821/20260520_131140_aBc123XyZ7.wav](https://minio.zoffice.vn/zipai/synthesized/vois_20260520_131005_4821/20260520_131140_aBc123XyZ7.wav)",
  "output_type": "minio"
}


```



```[cite: 1]

#### 💻 Ví dụ Code tích hợp:

**cURL (Nhận luồng Binary về file):**
```bash
curl -X 'POST' \
  'http://localhost:7009/synthesize_ID_v2' \
  -H 'Content-Type: multipart/form-data' \
  -F 'text=Hội đồng nhân dân tỉnh quyết định áp dụng công nghệ AI vào đời sống.' \
  -F 'id=vois_20260520_131005_4821' \
  -F 'output_type=binary' \
  --output ket_qua.wav

```

**Python (Nhận URL MinIO):**

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

### 3.3. Lấy danh sách các ID giọng nói đang có (`/list_ids`)

Liệt kê tất cả các định danh thư mục giọng nói (`id_directories`) đã được khởi tạo thành công và đang được lưu trữ trên hệ thống.

* **Phương thức:** `GET`

* **Đường dẫn:** `/list_ids`


#### 📤 Phản hồi mẫu (Response - JSON):

* **Mã trạng thái HTTP:** `200 OK`


```json
{
  "id_directories": [
    "vois_20260520_131005_4821",
    "vois_20260519_091522_1102"
  ]
}
```[cite: 1]

#### 💻 Ví dụ Code tích hợp:

**cURL:**
```bash
curl -X 'GET' 'http://localhost:7009/list_ids' -H 'accept: application/json'

```

**Python:**

```python
import requests

response = requests.get("http://localhost:7009/list_ids")
print(response.json())

```

---

### 3.4. Xóa một định danh giọng nói (`/delete_id/{timestamp}`)

Xóa bỏ hoàn toàn toàn bộ thư mục dữ liệu mẫu của một định danh giọng nói để giải phóng dung lượng dựa trên chuỗi thời gian timestamp của ID đó.

* **Phương thức:** `DELETE`

* **Đường dẫn:** `/delete_id/{timestamp}`

* **Tham số Đường dẫn (Path Parameter):**
* `timestamp`: Là phần chuỗi ký tự đứng sau tiền tố `vois_`.


* *Ví dụ thực tế:* Nếu ID đầy đủ là `vois_20260520_131005_4821`, thì phần `timestamp` cần truyền vào URL sẽ là `20260520_131005_4821`.





#### 📤 Phản hồi mẫu (Response - JSON):

* **Mã trạng thái HTTP:** `200 OK`


```json
{
  "message": "Directory with timestamp 20260520_131005_4821 deleted successfully."
}
```[cite: 1]

#### 💻 Ví dụ Code tích hợp:

**cURL:**
```bash
curl -X 'DELETE' 'http://localhost:7009/delete_id/20260520_131005_4821' -H 'accept: application/json'

```

**Python:**

```python
import requests

timestamp = "20260520_131005_4821"
response = requests.delete(f"http://localhost:7009/delete_id/{timestamp}")
print(response.json())

```

---

## 4. Quy trình xử lý dữ liệu ngầm (Pipeline nội bộ)

Để đảm bảo chất lượng giọng đọc nhân bản tự nhiên, mượt mà và tránh lỗi mô hình, hệ thống tự động thực hiện chuỗi tiền xử lý văn bản ngầm sau khi nhận yêu cầu:

1. **Chuyển đổi ngắt nghỉ dấu câu (`convert_punctuation`):** Hệ thống chỉ giữ lại dấu chấm (`.`) và dấu phẩy (`,`). Tất cả các ký tự đặc biệt phân tách câu khác như `;`, `:`, `-`, `(`, `)` đều được chuyển đổi thành dấu phẩy để tạo quãng ngắt nghỉ tự nhiên cho mô hình AI. Ký tự `&` được thay bằng chữ `và`.


2. **Xử lý mốc thời gian:** Tự động tìm kiếm các chuỗi định dạng ngày tháng/năm (Ví dụ: `2025-2026`, `20/05/2026`) để thêm dấu phẩy bao quanh, giúp mô hình đọc tách biệt mạch lạc, không bị dính chữ.


3. **Hạ chữ thường & Chuẩn hóa văn bản (`normalize_text` & `TTSnorm`):** Chuyển đổi toàn bộ chuỗi sang chữ thường, sau đó chạy qua bộ thư viện chuẩn hóa chữ viết tiếng Việt độc quyền để giải nghĩa chính xác từ viết tắt văn bản hành chính thông dụng và cách phát âm thuật ngữ tiếng Anh cơ bản (Ví dụ: *hđnd* -> *hội đồng nhân dân*, *AI* -> *Ây Ai*).


4. **Kiểm lỗi chính tả nâng cao (`correct_text`):** Gửi văn bản qua cổng dịch vụ `Spelling Correction` nội bộ để chỉnh sửa lỗi gõ sai chính tả trước khi đưa vào mô hình sinh âm thanh. Nếu dịch vụ này lỗi hoặc quá thời gian phản hồi (timeout > 5s), hệ thống sẽ tự động dùng văn bản gốc để tiếp tục quy trình nhằm đảm bảo tính sẵn sàng cao.



```


```
