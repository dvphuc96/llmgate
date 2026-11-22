# 0003. Rate limit token hai pha bằng token bucket có hoàn lại

Trạng thái: Chấp nhận · Ngày: 2026-11-22

## Bối cảnh
Số token output chỉ biết sau khi gọi xong. Nếu chỉ trừ sau, nhiều request đồng thời có thể vượt TPM rất xa. Nếu trừ `max_tokens` trước mà không hoàn lại, hạn mức bị lãng phí (nhiều client đặt `max_tokens` lớn).

## Quyết định
- Pha 1 trừ ước lượng `prompt_estimate + max_tokens` (mặc định 1024 nếu không có).
- Pha 2 điều chỉnh bằng chênh lệch thực tế: hoàn lại hoặc trừ thêm.
- Dùng token bucket của `ratelimitd` (hỗ trợ hits âm có chặn trần capacity), không dùng GCRA cho TPM.

## Hệ quả
- Không vượt TPM quá một request lớn nhất tại một thời điểm.
- Cần API `Adjust(key, delta)` trong `ratelimitd` — thêm vào dự án 1 (ghi issue liên dự án).
- Request bị client huỷ giữa chừng vẫn được điều chỉnh theo token đã sinh.
