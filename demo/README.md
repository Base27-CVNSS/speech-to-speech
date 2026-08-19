---
title: HF Realtime Voice
emoji: 🎙️
colorFrom: indigo
colorTo: purple
sdk: docker
app_port: 7860
pinned: false
short_description: Trò chuyện giọng nói thời gian thực qua WebSocket hoặc WebRTC với Speech-to-Speech
hf_oauth: true
hf_oauth_expiration_minutes: 10080
---

# 🎙️ Demo giọng nói thời gian thực

Giao diện hội thoại bằng giọng nói trên trình duyệt dành cho backend [`speech-to-speech`](https://github.com/Base27-CVNSS/speech-to-speech), sử dụng giao thức **OpenAI Realtime GA** qua **WebSocket** hoặc **WebRTC**.

Demo sử dụng `RealtimeSession` của gói `@openai/agents` đã khóa phiên bản. Các chức năng đặc thù của giao diện như hàng đợi, âm thanh, trực quan hóa, chọn thiết bị, camera và đo thời lượng được giữ bên ngoài lớp giao thức để không làm lẫn logic UI với Realtime API.

---

## 🚀 Chạy nhanh trên máy cục bộ

### 1. Khởi động backend Speech-to-Speech

Từ thư mục gốc của repository:

```bash
uv run speech-to-speech serve \
  --stt parakeet-tdt \
  --llm_backend transformers \
  --tts kokoro \
  --model_name "Qwen/Qwen3-4B-Instruct-2507" \
  --llm_device mps \
  --llm_torch_dtype float16 \
  --enable_live_transcription
```

Mặc định server Realtime lắng nghe tại:

```text
ws://localhost:8765/v1/realtime
```

Có thể đổi bằng `--host` và `--port`.

### 2. Cài dependency cho demo

```bash
npm ci --prefix demo
uv pip install -r demo/requirements.txt
```

Khai báo endpoint backend:

```bash
export SPEECH_TO_SPEECH_URL=ws://localhost:8765/v1/realtime
```

Các biến tùy chọn:

```bash
export SERPER_API_KEY=...    # tìm kiếm web; không có key thì tool bị tắt
export STARTUP_GREETING=...  # lời chào tự động; để rỗng để tắt
```

Khởi chạy ứng dụng:

```bash
uv run uvicorn --app-dir demo server:app --reload --port 7860
```

Mở:

```text
http://localhost:7860/
```

Nhấn vào vòng tròn trung tâm, cấp quyền microphone và bắt đầu nói.

> Trình duyệt yêu cầu **HTTPS hoặc `localhost`** để dùng `getUserMedia()` cho microphone/camera. HTTP thường trên địa chỉ LAN như `http://192.168.x.x` có thể bị chặn quyền thiết bị.

---

## 🐳 Chạy bằng Docker

```bash
docker build -t s2s-demo demo/
docker run \
  -p 7860:7860 \
  -e SPEECH_TO_SPEECH_URL=ws://host.docker.internal:8765/v1/realtime \
  s2s-demo
```

### Lưu ý Docker: WebSocket và WebRTC dùng namespace mạng khác nhau

- **WebRTC:** quá trình bắt tay SDP được proxy phía server, do đó `host.docker.internal` có thể trỏ tới máy host từ bên trong container.
- **WebSocket:** browser mở socket trực tiếp. Browser chạy trên máy host, nên `host.docker.internal` không nhất thiết là hostname hợp lệ ở phía browser.

Nếu muốn kiểm thử cả WebSocket lẫn WebRTC mà không phải đổi biến môi trường, cách đơn giản nhất là chạy demo trực tiếp bằng `uvicorn` thay vì Docker.

---

## 🔎 Kiểm tra nhanh backend

Có thể dùng `websocat`:

```bash
websocat ws://localhost:8765/v1/realtime
```

Nếu backend hoạt động, server sẽ trả về sự kiện `session.created` ngay sau khi kết nối.

---

## 🧠 Demo hoạt động như thế nào?

1. Adapter tạo một `RealtimeSession` bằng transport WebSocket hoặc WebRTC chính thức từ Agents SDK.
2. SDK gửi `session.update` theo schema OpenAI Realtime GA.
3. Với WebSocket, microphone được stream thành PCM16 24 kHz mono, đóng gói base64 bằng `input_audio_buffer.append`.
4. Browser giữ một bộ đệm giới hạn của các frame đã gửi để có thể phát lại đoạn giọng nói người dùng trong lịch sử hội thoại.
5. Backend trả về transcript và audio theo luồng.
6. Giao diện cập nhật trạng thái, hiệu ứng orb, lịch sử, tool call và âm thanh theo thời gian thực.

Backend phục vụ số phiên đồng thời phụ thuộc số pipeline được cấu hình bằng:

```text
--num_pipelines
```

---

## 🔌 WebSocket và WebRTC

### WebSocket

Phù hợp nhất khi:

- phát triển local;
- cần debug event JSON dễ dàng;
- cần phát lại bản ghi giọng nói người dùng trong lịch sử;
- dùng chế độ load balancer hiện tại.

Luồng âm thanh được gửi bằng event:

```text
input_audio_buffer.append
```

và nhận lại qua các audio delta.

### WebRTC

WebRTC có thể được chọn trong **Cài đặt → Transport** khi URL backend được cố định bằng biến môi trường.

Quy trình:

1. Browser tạo `RTCPeerConnection`.
2. Microphone được thêm dưới dạng media track.
3. SDP offer được POST tới `/api/calls`.
4. Demo proxy offer tới backend `POST /v1/realtime/calls`.
5. Sau khi bắt tay, audio RTP và data channel chạy trực tiếp giữa browser và backend.

Cài extra WebRTC cho backend:

```bash
pip install "speech-to-speech[webrtc]"
```

Nếu thiếu dependency, endpoint `/v1/realtime/calls` sẽ không hoạt động.

### Giới hạn WebRTC hiện tại

- Phát lại bản ghi người dùng trong lịch sử chỉ đầy đủ trên WebSocket.
- NAT phức tạp có thể cần STUN/TURN.
- Noise gate riêng của demo WebSocket không áp dụng nguyên trạng cho media track WebRTC.
- Snapshot camera phải nén nhỏ hơn do giới hạn message data-channel.
- Chế độ load balancer hiện ưu tiên WebSocket.

---

## 🌐 Ba chế độ kết nối backend

### 1. `SPEECH_TO_SPEECH_URL`

Đây là lựa chọn ưu tiên cho local/self-hosted:

```bash
export SPEECH_TO_SPEECH_URL=ws://localhost:8765/v1/realtime
```

Browser kết nối trực tiếp tới URL này. Khi được khai báo bằng môi trường, trường URL trong Settings sẽ bị khóa để tránh client tự đổi endpoint.

### 2. Không khai báo URL môi trường

Người dùng có thể nhập endpoint tại:

```text
Settings → Speech-to-speech server URL
```

Chế độ này phù hợp phát triển/thử nghiệm và chủ yếu dùng WebSocket.

### 3. `LOAD_BALANCER_URL`

Dành cho triển khai nhiều compute replica. Demo gọi `/api/session`, server proxy tới load balancer và nhận lại endpoint phiên cụ thể.

| `SPEECH_TO_SPEECH_URL` | `LOAD_BALANCER_URL` | Kết nối | URL UI | Transport | Metering |
|:---:|:---:|---|---|---|---|
| ✅ | bất kỳ | trực tiếp tới URL cố định | hiện, chỉ đọc | WS/WebRTC | tắt |
| – | – | trực tiếp tới URL người dùng | chỉnh sửa | WS | tắt |
| – | ✅ | qua LB proxy | ẩn | WS | tùy cấu hình |

---

## 👋 Lời chào khởi động

Khi tạo kết nối mới, demo có thể tạo một item ẩn yêu cầu model phát lời chào ngắn. Việc này đồng thời làm nóng prefix/prompt trước lượt nói đầu tiên.

Tùy chỉnh:

```bash
export STARTUP_GREETING="Hãy chào người dùng bằng tiếng Việt, ngắn gọn và tự nhiên."
```

Tắt hoàn toàn:

```bash
export STARTUP_GREETING=""
```

---

## 🧰 Công cụ trong hội thoại

Demo có thể bật/tắt công cụ trong giao diện.

### 🔎 Tìm kiếm web

Kết quả Google được truy vấn qua Serper.dev và proxy phía server để API key không lộ ra browser.

```bash
export SERPER_API_KEY=...
```

### 📷 Camera

Khi bật camera:

- browser hiển thị self-view;
- model có thể gọi tool chụp frame hiện tại;
- ảnh được gửi tới vision-language model để phân tích nội dung người dùng đang cho xem.

---

## 🎚️ Các thiết lập được lưu trong `localStorage`

| Thiết lập | Chức năng |
|---|---|
| Speech-to-speech server URL | endpoint Realtime WebSocket |
| Transport | WebSocket hoặc WebRTC |
| Microphone | thiết bị thu âm đầu vào |
| Speakers | thiết bị phát âm thanh đầu ra |
| Voice | giọng Qwen3-TTS |
| Instructions | system prompt gửi trong `session.update` |

Các khóa lưu trữ dùng namespace `s2s.ws.*`, cùng một số khóa âm thanh/transport riêng.

---

## 🔐 Quyền riêng tư và bảo mật

- Microphone/camera chỉ hoạt động khi người dùng cấp quyền trình duyệt.
- API key tìm kiếm web nên đặt phía server bằng biến môi trường/secret.
- Không nhúng khóa bí mật vào `main.js` hoặc HTML công khai.
- Khi dùng WebRTC proxy, backend đích phải được cố định từ môi trường; không biến proxy thành open proxy cho URL tùy ý.
- Triển khai Internet cần HTTPS, auth, rate limiting và chính sách CORS phù hợp.

---

## 📊 Hạn mức sử dụng khi triển khai

Metering chỉ bật trong cấu hình triển khai có load balancer/Space phù hợp. Local không tự áp hạn mức này.

Một số biến môi trường:

| Biến | Mặc định | Ý nghĩa |
|---|---:|---|
| `LIMIT_ANON_SEC` | `300` | số giây/ngày cho khách ẩn danh |
| `LIMIT_FREE_SEC` | `600` | số giây/ngày cho tài khoản miễn phí |
| `UNLIMITED_ORGS` | bổ sung | danh sách tổ chức được dùng không giới hạn |
| `USAGE_HASH_SECRET` | ngẫu nhiên | secret HMAC cho định danh/hạn mức |

---

## 🗂️ Các tệp chính

| Tệp | Vai trò |
|---|---|
| `index.html` | giao diện một trang, orb và modal cài đặt |
| `main.js` | state machine UI, settings, tools, camera, noise gate |
| `ui/chat.js` | lịch sử, bubble, transcript, tool streaming |
| `ui/account.js` | đăng nhập HF và UI hạn mức |
| `ui/dom.js` | helper DOM dùng chung |
| `auth.py` | HF OAuth và nhận diện người dùng |
| `limiter.py` | ngân sách thời lượng theo ngày bằng SQLite |
| `s2s-realtime-client.js` | adapter quanh `RealtimeSession` |
| `ws/codec.js` | chuyển đổi base64 ↔ PCM |
| `ws/user-audio-recorder.js` | bộ đệm và phát lại audio người dùng |
| `ws/orb-visualizer.js` | FFT → biến CSS của orb |
| `worklets/mic-capture.js` | AudioWorklet thu microphone |
| `worklets/audio-playback.js` | AudioWorklet phát audio assistant |
| `style.css` | bố cục, animation và dark theme |

---

## 🇻🇳 Gợi ý Việt hóa giao diện

Khi dịch UI, **không dịch** các giá trị kỹ thuật như:

```text
WebSocket
WebRTC
VAD
STT
TTS
LLM
session.update
response.create
input_audio_buffer.append
/v1/realtime
```

Chỉ dịch nhãn người dùng nhìn thấy, ví dụ:

| English | Tiếng Việt |
|---|---|
| Tap to start | Nhấn để bắt đầu |
| End | Kết thúc |
| Conversation | Hội thoại |
| Settings | Cài đặt |
| About | Giới thiệu |
| Microphone | Microphone |
| Speakers | Loa |
| Voice | Giọng nói |
| Instructions | Chỉ dẫn hệ thống |
| Restart | Kết nối lại |
| Join now | Tham gia ngay |
| Leave queue | Rời hàng đợi |

Cách này giữ nguyên tương thích giao thức nhưng mang lại trải nghiệm tiếng Việt tự nhiên và dễ bảo trì khi đồng bộ upstream.

---

## 📜 Nguồn và giấy phép

Demo là một phần của dự án Speech-to-Speech của Hugging Face. Bản Việt hóa tại `Base27-CVNSS/speech-to-speech` giữ nguyên cấu trúc kỹ thuật và ghi công upstream.

Xem giấy phép tại [`../LICENSE`](../LICENSE).
