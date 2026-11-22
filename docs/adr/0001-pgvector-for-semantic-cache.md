# 0001. Dùng pgvector cho semantic cache thay vì vector DB riêng

Trạng thái: Chấp nhận · Ngày: 2026-11-22

## Bối cảnh
Semantic cache cần: tìm top-1 theo cosine, **lọc chặt theo tenant + alias + context_hash**, TTL, xoá theo tenant. Quy mô: hàng trăm nghìn đến vài triệu vector.

## Các lựa chọn
| Lựa chọn | Ưu | Nhược |
|---|---|---|
| **pgvector (HNSW)** | Đã có Postgres; lọc + vector trong một SQL; giao dịch, backup chung | Lọc rất chọn lọc có thể giảm recall HNSW (cần `ef_search` cao hoặc partial index) |
| Redis Stack (RediSearch vector) | Nhanh, cùng Redis | Thêm module, bộ nhớ RAM đắt |
| Qdrant / Weaviate | Lọc payload tốt | Thêm một hệ thống vận hành |

## Quyết định
pgvector. Bật `hnsw.iterative_scan` (pgvector ≥ 0.8) để lọc tenant không làm mất kết quả. Đo latency trong benchmark; nếu p99 tra cứu > 20 ms ở 1M vector thì xem xét partition theo tenant lớn.

## Hệ quả
- Ít thành phần; cô lập tenant đảm bảo bằng `WHERE tenant_id = $1` (và test bắt buộc).
- Tải ghi cache đè lên Postgres chung với usage ⇒ ghi cache bất đồng bộ, có thể bỏ khi quá tải.
