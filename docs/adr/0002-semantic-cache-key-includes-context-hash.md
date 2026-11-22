# 0002. Khoá semantic cache = embedding câu cuối + hash ngữ cảnh chính xác

Trạng thái: Chấp nhận · Ngày: 2026-11-22

## Bối cảnh
"Nó bao nhiêu tiền?" trong hai cuộc hội thoại khác nhau có embedding giống hệt nhưng ý khác hoàn toàn. Embed cả hội thoại thì tốn và độ tương đồng bị pha loãng.

## Các lựa chọn
1. Embed chỉ câu cuối — hit cao, **sai nghiêm trọng** khi có ngữ cảnh.
2. Embed toàn bộ hội thoại — đắt, hit thấp, vẫn có thể trùng sai.
3. **Embed câu cuối + yêu cầu `context_hash` (system prompt + lịch sử) khớp chính xác.**

## Quyết định
Lựa chọn 3. Trên thực tế semantic cache hiệu quả nhất ở câu hỏi một lượt (FAQ, tìm kiếm, phân loại) với system prompt cố định — đúng nơi lựa chọn 3 cho hit.

## Hệ quả
- Hội thoại nhiều lượt gần như không hit semantic (chấp nhận; đúng hơn là quan trọng hơn).
- Ứng dụng muốn hit nhiều hơn phải giữ system prompt ổn định (ghi trong README hướng dẫn tích hợp).
