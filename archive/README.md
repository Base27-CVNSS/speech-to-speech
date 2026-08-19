# 🗃️ Các mô hình đã lưu trữ

Thư mục này chứa các triển khai mô hình **đã ngừng được nối trực tiếp vào pipeline chính**, nhưng vẫn được giữ lại trong repository để tham khảo, thử nghiệm hoặc phục hồi khi cần.

Các thành phần hiện có:

- **STT — Moonshine:** `archive/STT/moonshine_handler.py`
- **TTS — Parler:** `archive/TTS/parler_handler.py`
- **TTS — Melo:** `archive/TTS/melo_handler.py`
- **Tham số CLI cũ của Parler TTS:** `archive/arguments_classes/parler_tts_arguments.py`
- **Tham số CLI cũ của Melo TTS:** `archive/arguments_classes/melo_tts_arguments.py`

Các backend này không còn nằm trong dependency mặc định và không còn được nối vào pipeline/CLI hiện hành.

Nếu muốn chạy thủ công một backend trong `archive/`, bạn cần:

1. tự cài dependency tương ứng;
2. kiểm tra khả năng tương thích với phiên bản Python/PyTorch hiện tại;
3. tự nối handler vào pipeline hoặc viết script thử nghiệm riêng;
4. không giả định API cũ vẫn tương thích với kiến trúc Realtime mới.

> `archive/` nên được xem là **mã tham khảo/legacy**, không phải tập backend được hỗ trợ chính thức ở phiên bản hiện tại.
