<div align="center">
  <div>&nbsp;</div>
  <img src="https://raw.githubusercontent.com/huggingface/speech-to-speech/main/logo.png" width="600" alt="Speech-to-Speech"/>

# 🎙️ Speech-to-Speech — Xây dựng trợ lý giọng nói thời gian thực bằng mô hình mã nguồn mở

[![PyPI](https://img.shields.io/pypi/v/speech-to-speech)](https://pypi.org/project/speech-to-speech/)
[![Python](https://img.shields.io/pypi/pyversions/speech-to-speech)](https://pypi.org/project/speech-to-speech/)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](./LICENSE)
[![OpenAI Realtime](https://img.shields.io/badge/API-OpenAI%20Realtime-412991)](./src/speech_to_speech/api/openai_realtime/README.md)
[![Vietnamese](https://img.shields.io/badge/Tài%20liệu-Tiếng%20Việt-da251d)](https://github.com/Base27-CVNSS/speech-to-speech)

**Pipeline hội thoại giọng nói độ trễ thấp, mô-đun hóa hoàn toàn:**  
`VAD → STT → LLM → TTS`

</div>

> **Bản Việt hóa chuyên nghiệp** của kho mã `huggingface/speech-to-speech`, được duy trì tại `Base27-CVNSS/speech-to-speech`. Phần mã nguồn, giao thức, tên tham số CLI và API được giữ nguyên để bảo đảm tương thích với upstream. Dự án gốc thuộc Hugging Face và các tác giả tương ứng; giấy phép hiện tại là **Apache License 2.0**.

---

## 📌 Speech-to-Speech là gì?

Speech-to-Speech là một **khung xây dựng tác tử/trợ lý giọng nói thời gian thực** theo kiến trúc ghép tầng. Thay vì buộc người dùng vào một mô hình end-to-end duy nhất, hệ thống chia quá trình hội thoại thành bốn mô-đun độc lập:

1. **VAD — Voice Activity Detection:** phát hiện khi người dùng bắt đầu/kết thúc nói.
2. **STT — Speech-to-Text:** chuyển tiếng nói thành văn bản.
3. **LLM — Large Language Model:** hiểu ngữ cảnh, suy luận, gọi công cụ và tạo câu trả lời.
4. **TTS — Text-to-Speech:** tổng hợp câu trả lời thành giọng nói và phát luồng âm thanh trở lại phía người dùng.

Mỗi mô-đun có thể thay thế bằng backend khác. LLM sử dụng giao thức tương thích OpenAI nên có thể kết nối dịch vụ đám mây, Hugging Face Inference Providers, OpenRouter, vLLM, llama.cpp hoặc mô hình chạy trực tiếp trên máy.

Pipeline cung cấp tập sự kiện lõi của **OpenAI Realtime API** qua **WebSocket** và **WebRTC**, giúp ứng dụng hiện có có thể chuyển endpoint sang server tự host mà không phải thiết kế lại toàn bộ tầng giao tiếp.

---

## ✨ Điểm nổi bật

- ⚡ **Độ trễ thấp:** xử lý theo luồng, phù hợp hội thoại thời gian thực.
- 🧩 **Mô-đun hóa:** thay VAD, STT, LLM hoặc TTS độc lập.
- 🔌 **Tương thích OpenAI Realtime:** hỗ trợ WebSocket và WebRTC trên endpoint `/v1/realtime`.
- 🖥️ **Local-first:** có thể chạy nhiều thành phần hoàn toàn trên máy cá nhân.
- 🌐 **Linh hoạt backend LLM:** OpenAI, HF Inference Providers, OpenRouter, vLLM, llama.cpp và các API tương thích OpenAI.
- 🗣️ **Đa ngôn ngữ:** phạm vi ngôn ngữ phụ thuộc STT/TTS được chọn; hỗ trợ cấu hình `--language` hoặc `--language auto`.
- 🧠 **Smart Turn:** cải thiện phát hiện kết thúc lượt nói, giảm cắt lời sai và hỗ trợ barge-in.
- 🧰 **Tool calling:** LLM có thể gọi công cụ trong luồng hội thoại.
- 🐳 **Docker:** hỗ trợ triển khai bằng Docker Compose.
- 🍎 **Apple Silicon:** hỗ trợ MLX/MLX-Audio/MLX-LM.
- 🐧 **CUDA / CPU:** hỗ trợ nhiều backend trên Linux và máy không có GPU chuyên dụng.

---

## 🧱 Kiến trúc tổng quát

```text
┌──────────────┐
│ Microphone   │
│  Người dùng  │
└──────┬───────┘
       │ PCM / audio stream
       ▼
┌──────────────────────┐
│ VAD / Smart Turn     │
│ Phát hiện lượt nói   │
└─────────┬────────────┘
          │ speech segment
          ▼
┌──────────────────────┐
│ STT                  │
│ Speech → Text        │
└─────────┬────────────┘
          │ transcript
          ▼
┌──────────────────────┐
│ LLM                  │
│ Text / Tool Calling  │
└─────────┬────────────┘
          │ streamed text
          ▼
┌──────────────────────┐
│ TTS                  │
│ Text → Speech        │
└─────────┬────────────┘
          │ audio stream
          ▼
┌──────────────┐
│ Speaker      │
│  Người dùng  │
└──────────────┘
```

### Luồng thời gian thực

```text
Client
  │
  ├─ input_audio_buffer.append
  ├─ session.update
  ├─ conversation.item.create
  ├─ response.create / response.cancel
  │
  ▼
OpenAI Realtime-compatible Server
  │
  ├─ VAD + Smart Turn
  ├─ STT streaming
  ├─ LLM streaming + tools
  └─ TTS streaming
  │
  ▼
Client nhận transcript + audio delta + response.done
```

> Server triển khai **tập lõi** của giao thức Realtime; đây không phải tuyên bố tương thích 100% với toàn bộ OpenAI Realtime API.

---

## 🚀 Khởi động nhanh

### 1. Cài đặt

Yêu cầu **Python 3.10+**.

```bash
pip install speech-to-speech
```

### 2. Khởi chạy server

```bash
export OPENAI_API_KEY=...
speech-to-speech serve
```

Mặc định server lắng nghe tại:

```text
ws://localhost:8765/v1/realtime
```

### 3. Trò chuyện từ terminal thứ hai

```bash
speech-to-speech talk --url ws://127.0.0.1:8765/v1/realtime
```

### 4. Chạy server + microphone/speaker bằng một lệnh

```bash
speech-to-speech local
```

---

## 🇻🇳 Cấu hình gợi ý cho tiếng Việt

Pipeline không khóa ngôn ngữ. Chất lượng tiếng Việt phụ thuộc vào STT, LLM và TTS bạn ghép với nhau.

Một cấu hình thực dụng là dùng Whisper cho nhận dạng tiếng Việt và Qwen3-TTS cho tổng hợp đa ngôn ngữ:

```bash
speech-to-speech serve \
    --stt whisper \
    --stt_model_name openai/whisper-large-v3-turbo \
    --language vi \
    --llm_backend responses-api \
    --tts qwen3 \
    --qwen3_tts_language auto \
    --enable_live_transcription
```

Nếu muốn tự động nhận diện ngôn ngữ giữa tiếng Việt và các ngôn ngữ khác:

```bash
speech-to-speech serve \
    --stt whisper \
    --language auto \
    --enable_lang_prompt \
    --llm_backend responses-api \
    --tts qwen3
```

**Khuyến nghị:** khi ưu tiên tiếng Việt, hãy kiểm thử riêng ba lớp STT → LLM → TTS. Một pipeline đa ngôn ngữ không đồng nghĩa mọi backend đều có chất lượng tiếng Việt giống nhau.

---

## 🧩 Các thành phần được hỗ trợ

| Thành phần | Backend | Nền tảng | Ghi chú |
|---|---|---|---|
| VAD | Silero VAD v5 | đa nền tảng | tích hợp sẵn |
| STT | Parakeet TDT | CUDA / CPU / Apple Silicon | mặc định |
| STT | Whisper | CUDA / CPU | tích hợp sẵn |
| STT | Faster Whisper | CUDA / CPU | extra `faster-whisper` |
| STT | Lightning Whisper MLX | Apple Silicon | extra `whisper-mlx` |
| STT | MLX Audio Whisper | Apple Silicon | tích hợp trên macOS |
| STT | Paraformer / FunASR | CUDA / CPU | extra `paraformer` |
| LLM | Responses API | cloud / self-hosted | mặc định |
| LLM | Chat Completions | cloud / self-hosted | tương thích OpenAI |
| LLM | Transformers | CUDA / CPU | chạy cục bộ |
| LLM | mlx-lm | Apple Silicon | chạy cục bộ |
| TTS | Qwen3-TTS | GGML / CUDA / MLX | mặc định |
| TTS | Kokoro-82M | CUDA / CPU / Apple Silicon | tùy nền tảng |
| TTS | Pocket TTS | CPU / CUDA | hỗ trợ streaming |
| TTS | ChatTTS | CUDA / CPU | tiếng Anh/Trung |
| TTS | MMS TTS | CUDA / CPU | nhiều checkpoint ngôn ngữ |

Chọn backend bằng:

```text
--stt
--llm_backend
--tts
```

Xem toàn bộ tham số:

```bash
speech-to-speech serve -h
speech-to-speech talk -h
```

---

## 📦 Thành phần tùy chọn

```bash
pip install "speech-to-speech[kokoro]"
pip install "speech-to-speech[pocket]"
pip install "speech-to-speech[chattts]"
pip install "speech-to-speech[faster-whisper]"
pip install "speech-to-speech[whisper-mlx]"
pip install "speech-to-speech[paraformer]"
pip install "speech-to-speech[mlx-lm]"
```

Các triển khai đã ngừng dùng như MeloTTS được lưu trong [`archive/`](./archive) và không còn nối trực tiếp vào CLI hiện tại.

---

## 🧭 Ba lệnh CLI chính

| Lệnh | Chức năng | Dùng khi |
|---|---|---|
| `speech-to-speech serve` | chạy server Realtime WebSocket/WebRTC | xây app, robot hoặc thiết bị khách |
| `speech-to-speech talk --url ...` | chạy client microphone/speaker đóng gói sẵn | kết nối tới một server đang có |
| `speech-to-speech local` | chạy server + client trong cùng tiến trình | thử nghiệm nhanh trên một máy |

Server mặc định bind vào `127.0.0.1`. Muốn truy cập từ máy khác trong mạng:

```bash
speech-to-speech serve --host 0.0.0.0
```

> Chỉ mở server ra mạng tin cậy hoặc đặt phía sau gateway có xác thực.

---

## 🔌 OpenAI Realtime API

Ví dụ kết nối bằng OpenAI SDK:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8765/v1",
    websocket_base_url="ws://localhost:8765/v1",
    api_key="not-needed",
)

with client.realtime.connect(model="local") as conn:
    conn.send(
        {
            "type": "session.update",
            "session": {
                "type": "realtime",
                "instructions": "Bạn là một trợ lý tiếng Việt hữu ích.",
                "audio": {
                    "input": {
                        "turn_detection": {
                            "type": "server_vad",
                            "interrupt_response": True,
                        }
                    }
                },
            },
        }
    )

    for event in conn:
        print(event.type)
```

Các sự kiện đầu vào lõi gồm:

```text
input_audio_buffer.append
session.update
conversation.item.create
conversation.item.truncate
response.create
response.cancel
```

Phía server có thể stream sự kiện bắt đầu/kết thúc nói, transcript, audio delta, tool call và `response.done`.

Tài liệu chi tiết: [`src/speech_to_speech/api/openai_realtime/README.md`](./src/speech_to_speech/api/openai_realtime/README.md).

---

## 🧠 Backend LLM

LLM thường là thành phần tốn tài nguyên và ảnh hưởng độ trễ nhất. Dự án hỗ trợ ba hướng triển khai:

### 1. API nhà cung cấp

- OpenAI
- Hugging Face Inference Providers
- OpenRouter
- các dịch vụ tương thích OpenAI khác

### 2. Server tự host

- vLLM
- llama.cpp
- server tương thích `/v1/responses`
- server tương thích `/v1/chat/completions`

### 3. Chạy trực tiếp trong tiến trình

- Transformers trên CUDA / CPU
- mlx-lm trên Apple Silicon

Hai backend API chính:

```text
--llm_backend responses-api      → /v1/responses
--llm_backend chat-completions   → /v1/chat/completions
```

---

## 🏠 Chạy LLM hoàn toàn cục bộ với llama.cpp

Terminal 1:

```bash
llama-server \
    -hf ggml-org/gemma-4-E4B-it-GGUF \
    -np 2 \
    -c 65536 \
    -fa on \
    --swa-full
```

Terminal 2:

```bash
speech-to-speech serve \
    --stt parakeet-tdt \
    --llm_backend responses-api \
    --tts qwen3 \
    --model_name "ggml-org/gemma-4-E4B-it-GGUF" \
    --responses_api_base_url "http://127.0.0.1:8080/v1" \
    --responses_api_api_key "" \
    --responses_api_stream \
    --enable_live_transcription
```

---

## 📴 Chạy offline

Sau khi đã cài dependency và tải/cached đầy đủ model cần thiết, pipeline có thể chạy khi không có Internet.

Nên chạy đúng cấu hình một lần khi còn online để cache:

- STT model
- LLM model
- TTS model
- Silero VAD
- NLTK resource
- Smart Turn checkpoint

Sau đó có thể bật chế độ offline của Hugging Face Hub:

```bash
HF_HUB_OFFLINE=1 speech-to-speech serve \
    --model_name "ggml-org/gemma-4-E4B-it-GGUF" \
    --responses_api_base_url "http://127.0.0.1:8080/v1" \
    --responses_api_api_key ""
```

Nếu dùng Smart Turn và muốn phụ thuộc hoàn toàn vào file cục bộ:

```bash
--smart_turn_model_path /path/to/smart-turn-v3.2-cpu.onnx
```

Hoặc tắt:

```bash
--no_smart_turn
```

---

## 🌍 Hỗ trợ đa ngôn ngữ

Phạm vi ngôn ngữ phụ thuộc backend:

| Thành phần | Backend | Phạm vi |
|---|---|---|
| STT | Parakeet TDT | 25 ngôn ngữ châu Âu |
| STT | Whisper / Faster Whisper / Whisper MLX | đa ngôn ngữ, tùy checkpoint |
| STT | Paraformer | tùy checkpoint FunASR; mặc định thiên về tiếng Trung |
| TTS | Qwen3-TTS | đa ngôn ngữ; mặc định `auto` |
| TTS | Kokoro | nhiều mapping ngôn ngữ/giọng |
| TTS | ChatTTS | tiếng Anh và tiếng Trung |
| TTS | MMS TTS | nhiều ngôn ngữ qua checkpoint MMS |

Hai cách dùng phổ biến:

```bash
# Cố định một ngôn ngữ
--language vi

# Tự phát hiện ngôn ngữ từng lượt nói
--language auto
```

Có thể thêm:

```bash
--enable_lang_prompt
```

để nhắc LLM trả lời theo ngôn ngữ vừa phát hiện.

---

## 🍎 Apple Silicon

Thiết lập tối ưu nhanh:

```bash
speech-to-speech local --mac-optimal-settings
```

Preset này ưu tiên các backend phù hợp Apple Silicon và MLX, nhưng mọi tham số bạn truyền rõ ràng như `--stt`, `--tts`, `--llm_backend`, `--model_name` vẫn được ưu tiên hơn preset.

---

## 🐳 Docker

Cài NVIDIA Container Toolkit rồi chạy:

```bash
docker compose up
```

Docker Compose hiện khởi động llama.cpp cùng server Realtime, sử dụng các cổng chính:

```text
8080  → llama.cpp / LLM
8765  → Speech-to-Speech Realtime server
```

---

## 🎛️ Smart Turn và ngắt lời tự nhiên

Silero VAD chịu trách nhiệm phát hiện đoạn giọng nói. Smart Turn có thể xác minh quyết định kết thúc lượt bằng nội dung/ngữ điệu, nhờ đó giảm trường hợp hệ thống trả lời quá sớm khi người dùng chỉ đang tạm ngừng giữa câu.

Các tham số đáng chú ý:

```text
--thresh
--min_speech_ms
--min_speech_continuation_ms
--min_silence_ms
--short_segment_merge_ms
--speculative_reopen_ms
--smart_turn_threshold
--smart_turn_max_wait_ms
--no_smart_turn
```

Cơ chế “reopen” cho phép một lượt vừa bị coi là kết thúc được mở lại nếu người dùng nói tiếp trong cửa sổ ngắn, đồng thời loại bỏ phản hồi suy đoán cũ trước khi nó đến tai người dùng.

---

## 🧰 Tool calling

Client đóng gói sẵn có thể nạp công cụ Python cục bộ:

```bash
speech-to-speech talk \
    --url ws://127.0.0.1:8765/v1/realtime \
    --tool-module your_tools
```

Hoặc:

```bash
speech-to-speech local --tool-module your_tools
```

Thiết kế tool calling chi tiết nằm trong tài liệu Realtime Engine.

---

## 🔐 Lưu ý bảo mật

- Server cơ bản **không tự cung cấp cơ chế xác thực và throttling toàn diện** cho mọi đường triển khai.
- Không nên bind `0.0.0.0` ra Internet công cộng nếu chưa có reverse proxy/gateway, TLS, auth và rate limit.
- API key nên nằm phía server; không nhúng khóa bí mật vào frontend công khai.
- Khi bật LLM proxy, chỉ nên dùng trong mạng tin cậy hoặc sau gateway kiểm soát truy cập.

---

## 🛠️ Phát triển từ mã nguồn

```bash
git clone https://github.com/Base27-CVNSS/speech-to-speech.git
cd speech-to-speech
uv sync
```

Kiểm thử:

```bash
pytest
ruff check
```

---

## 🗂️ Cấu trúc khái niệm

```text
speech-to-speech/
├── src/speech_to_speech/
│   ├── STT/                  # nhận dạng giọng nói
│   ├── TTS/                  # tổng hợp giọng nói
│   ├── LLM/                  # mô hình ngôn ngữ / backend API
│   ├── VAD/                  # phát hiện hoạt động giọng nói
│   ├── api/openai_realtime/  # Realtime WebSocket/WebRTC
│   └── arguments_classes/    # cấu hình CLI
├── demo/                     # demo web thời gian thực
├── examples/                 # ví dụ triển khai
├── archive/                  # backend cũ/đã ngừng dùng
├── docs/                     # tài liệu và tài nguyên
├── Dockerfile
├── docker-compose.yml
└── pyproject.toml
```

---

## 🎯 Ứng dụng

Speech-to-Speech phù hợp để xây dựng:

- 🤖 robot giao tiếp bằng giọng nói;
- 🧑‍💻 trợ lý AI desktop/local;
- 📞 tổng đài hội thoại AI;
- 🏠 trợ lý nhà thông minh;
- 🚘 giao diện giọng nói cho thiết bị biên;
- 🎓 trợ giảng AI nói chuyện trực tiếp;
- ♿ giao diện hỗ trợ tiếp cận bằng giọng nói;
- 🧪 nền tảng nghiên cứu VAD/STT/LLM/TTS và turn-taking;
- 🔒 hệ thống hội thoại riêng tư chạy local/offline.

---

## ⚖️ Ưu điểm và giới hạn

| Ưu điểm | Giới hạn |
|---|---|
| thay từng mô-đun độc lập | cần tinh chỉnh nhiều thành phần để đạt độ trễ thấp |
| dễ tự host | model lớn đòi hỏi RAM/VRAM đáng kể |
| không khóa vào một nhà cung cấp | chất lượng phụ thuộc tổ hợp STT + LLM + TTS |
| tương thích giao thức Realtime lõi | chưa phải toàn bộ OpenAI Realtime API |
| hỗ trợ local/offline | cần tải/cached model trước |
| đa ngôn ngữ | mức hỗ trợ khác nhau theo backend |

---

## 🧾 Bản chất kỹ thuật

Speech-to-Speech **không phải một mô hình AI duy nhất**. Nó là một **orchestrator/pipeline hội thoại thời gian thực** nối các mô hình chuyên biệt bằng queue, streaming và state của phiên hội thoại.

Có thể hình dung:

```text
Speech-to-Speech = Realtime Orchestration Layer
                 + VAD / Turn Detection
                 + STT Adapter
                 + LLM Adapter
                 + TTS Adapter
                 + OpenAI-compatible Realtime Transport
```

Điểm mạnh lớn nhất nằm ở **khả năng thay engine mà không phải viết lại toàn bộ hệ thống hội thoại**.

---

## 🤝 Đóng góp

Issue và Pull Request được hoan nghênh. Với thay đổi lớn, nên mở issue trước để trao đổi hướng thiết kế.

Upstream:

- `huggingface/speech-to-speech`

Bản Việt hóa:

- `Base27-CVNSS/speech-to-speech`

Khi đồng bộ upstream, nên ưu tiên:

1. giữ nguyên API/CLI;
2. tránh dịch tên class, event, tham số và biến cấu hình;
3. chỉ Việt hóa tài liệu, nhãn giao diện và thông báo dành cho người dùng;
4. kiểm thử lại Realtime, Docker và CLI sau mỗi lần merge.

---

## 📜 Giấy phép và ghi công

Kho mã hiện sử dụng **Apache License 2.0**. Xem [`LICENSE`](./LICENSE).

Bản Việt hóa không thay đổi quyền tác giả của dự án gốc. Khi sử dụng trong nghiên cứu hoặc sản phẩm, hãy ghi công dự án Speech-to-Speech và các mô hình thành phần tương ứng như Silero VAD, Parakeet/Whisper, LLM và TTS mà bạn thực tế sử dụng.

---

<div align="center">

**🎙️ Nói → Hiểu → Suy luận → Trả lời bằng giọng nói**

`VAD → STT → LLM → TTS`

</div>
