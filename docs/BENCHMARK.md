# BENCHMARK — llmgate

## 0. Môi trường
| CPU | RAM | Docker | Postgres/pgvector | Redis | Embedding model | Commit |
|---|---|---|---|---|---|---|

## 1. Overhead và streaming (mock provider)
| Kịch bản | RPS / VU | p50 ms | p95 | p99 | Overhead p99 | Lỗi % | RAM | CPU |
|---|---|---|---|---|---|---|---|---|
| L1 overhead | | | | | | | | |
| L2 exact hit | | | | | | | | |
| L3 1.000 stream | | | | | | | | |
| L4 hỗn hợp | | | | | | | | |

## 2. Eval semantic cache
| Ngưỡng | Precision EN | Recall EN | Precision VI | Recall VI | F0.5 |
|---|---|---|---|---|---|
| 0.85 | | | | | |
| 0.90 | | | | | |
| 0.93 | | | | | |
| 0.95 | | | | | |
| 0.97 | | | | | |
**Ngưỡng chọn:** … vì …

## 3. Tiết kiệm chi phí (workload 5.000 request)
| Cấu hình | Gọi provider | Hit exact % | Hit semantic % | Tổng USD | Tiết kiệm % | p50 ms | p99 ms | Precision chấm tay |
|---|---|---|---|---|---|---|---|---|
| no-cache | | – | – | | – | | | – |
| exact | | | – | | | | | – |
| exact+semantic | | | | | | | | |

## 4. Chaos
| ID | Thành công % | Thời gian breaker mở | p99 | Ghi chú |
|---|---|---|---|---|
| K1 | | | | |
| K2 | | | | |
| K3 | | | | |
| K4 | | | | |
| K5 | | | | |
| K6 | | | | |

## 5. Chi phí trên mỗi khách hàng (ví dụ dữ liệu mô phỏng)
| Tenant | Gói (USD/tháng) | Request | Chi phí LLM USD | Biên lợi nhuận gộp % |
|---|---|---|---|---|
