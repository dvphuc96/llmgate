# 0004. Không fallback sau khi đã gửi byte đầu tiên cho client

Trạng thái: Chấp nhận · Ngày: 2026-11-22

## Bối cảnh
Khi đang stream, provider có thể ngắt giữa chừng. Gateway có thể gọi provider khác, nhưng client đã nhận một nửa câu trả lời từ model A.

## Các lựa chọn
1. Fallback và nối tiếp — câu trả lời ghép từ hai model, có thể mâu thuẫn, trùng lặp.
2. Buffer toàn bộ rồi mới gửi — mất lợi ích streaming (TTFT tăng mạnh).
3. **Chỉ fallback trước byte đầu; sau đó gửi chunk lỗi chuẩn và đóng stream.**

## Quyết định
Lựa chọn 3. Timeout TTFT (mặc định 10 s) là điểm fallback chính cho stream.

## Hệ quả
- Client cần xử lý lỗi giữa stream (SDK OpenAI đã có).
- Metric `llm_stream_aborted_total{provider}` để theo dõi; nếu cao, cân nhắc chế độ buffer tuỳ chọn cho tenant cần độ tin cậy hơn độ trễ.
