# Tài liệu Hướng dẫn Sử dụng API Speech-to-Text

Tài liệu này hướng dẫn tích hợp và sử dụng API chuyển đổi giọng nói thành văn bản (Speech-to-Text) được cung cấp tại `stt1.zipai.vn`.

---

## 1. Tổng quan

API nhận đầu vào là một file âm thanh và trả về nội dung văn bản được nhận dạng từ giọng nói trong file đó, kèm theo thông báo trạng thái xử lý.

- **Base URL:** `https://stt1.zipai.vn`
- **Định dạng phản hồi:** JSON với kết quả nhận dạng và thời gian xử lý

---

## 2. Endpoints

### 2.1. Chuyển đổi giọng nói thành văn bản — `POST /speech_to_text`

Nhận một file âm thanh và trả về nội dung văn bản được nhận dạng.

- **Content-Type:** `multipart/form-data`

#### Tham số Request Body

| Tham số | Kiểu | Bắt buộc | Mô tả |
| :--- | :--- | :---: | :--- |
| `audio_file` | File (Binary) | ✅ | File âm thanh cần nhận dạng (ví dụ: `.wav`, `.mp3`,...). |

#### Phản hồi mẫu — `200 OK`

```json
{
  "recognized_text": "vũ đức huy chính chính ocong i huyaho vn",
  "message": "Xử lý thành công"
}
```

| Trường | Kiểu | Mô tả |
| :--- | :--- | :--- |
| `recognized_text` | String | Nội dung văn bản được nhận dạng từ file âm thanh. |
| `message` | String | Thông báo trạng thái xử lý. |

#### Ví dụ tích hợp

**cURL:**

```bash
curl -X POST 'https://stt1.zipai.vn/speech_to_text' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'audio_file=@recording.wav;type=audio/wav'
```

**Python:**

```python
import requests

url = "https://stt1.zipai.vn/speech_to_text"
files = {"audio_file": ("recording.wav", open("recording.wav", "rb"), "audio/wav")}

response = requests.post(url, files=files)
print(response.json())
```

**JavaScript (fetch):**

```javascript
const formData = new FormData();
formData.append("audio_file", fileInput.files[0]);

const response = await fetch("https://stt1.zipai.vn/speech_to_text", {
  method: "POST",
  headers: { "accept": "application/json" },
  body: formData
});

const data = await response.json();
console.log(data.recognized_text);
```

---

## 3. Mã trạng thái HTTP

| Mã | Ý nghĩa |
| :--- | :--- |
| `200 OK` | Xử lý thành công, trả về văn bản nhận dạng. |
| `400 Bad Request` | Thiếu tham số bắt buộc hoặc định dạng file không hợp lệ. |
| `500 Internal Server Error` | Lỗi phía máy chủ trong quá trình xử lý. |

---

## 4. Lưu ý

- File âm thanh nên có chất lượng rõ ràng, ít tạp âm để đạt độ chính xác nhận dạng cao nhất.
- Định dạng `.wav` được khuyến nghị để tránh bước chuyển đổi ngầm phía server.
- API xử lý đồng bộ — client cần chờ đến khi nhận được phản hồi đầy đủ.
