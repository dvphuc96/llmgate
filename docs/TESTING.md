# TESTING — llmgate

## 1. Tầng test

| Tầng | Công cụ | Trọng tâm |
|---|---|---|
| Unit | `testing`, httptest | Adapter mapping, SSE parser, chuẩn hoá khoá cache, tính chi phí, router |
| Contract | Bản ghi phản hồi thật (golden file) của từng provider | Adapter không vỡ khi đọc phản hồi thật |
| Compatibility | Python `openai` SDK + Node `openai` SDK trỏ vào gateway | Drop-in thật sự |
| Integration | testcontainers (Postgres+pgvector, Redis) | Budget, cache cô lập tenant, usage ledger |
| Eval | `eval/` (Go + CSV) | Precision/recall semantic cache |
| Load | k6 + mock provider | N1, N2 |
| Chaos | Mock lỗi, Toxiproxy | Failover, Redis/Postgres chết |

## 2. Unit test bắt buộc

| Test | Kỳ vọng |
|---|---|
| `TestAnthropicAdapter_SystemMessageMapping` | message `system` OpenAI → trường `system` |
| `TestAnthropicAdapter_StreamEventsToChunks` | `content_block_delta` → `choices[0].delta.content` |
| `TestGeminiAdapter_RoleMapping` | `assistant` → `model` |
| `TestErrorMapping_RetryableClassification` | bảng §5 SPEC |
| `TestExactKey_IgnoresUserAndStreamFields` | |
| `TestExactKey_DiffersByTenant` | |
| `TestCost_MicroUSDRounding` | không float drift; tổng 1M request khớp |
| `TestRouter_WeightedDistribution` | 10k lần chọn ≈ 70/30 ± 2 % |
| `TestRouter_NoFallbackAfterFirstByte` | |
| `TestTPM_RefundAfterActualUsage` | |
| `TestLogs_NoPromptNoProviderKey` | |

## 3. Bảo mật / cô lập (bắt buộc pass)

- `TestSemanticCache_NeverCrossTenant`: tenant A và B hỏi cùng một câu ⇒ B không bao giờ nhận cache của A.
- `TestSemanticCache_DifferentSystemPromptNoHit`: cùng câu hỏi, system prompt khác ⇒ miss.
- `TestRevokedKeyRejectedWithin30s`.
- `TestModelNotAllowedForKey`.

## 4. Eval semantic cache

```
Dữ liệu: eval/pairs.csv  (q1, q2, label∈{same,different}, lang)
  - 1.700 cặp tiếng Anh (mẫu Quora Question Pairs, cân bằng nhãn)
  - 300 cặp tiếng Việt tự viết, chủ đề CSKH (đổi mật khẩu, hoàn tiền, giao hàng…)
Quy trình:
  embed q1, q2 bằng model embedding đã chọn → cosine
  với threshold t ∈ {0.80, 0.82, …, 0.99}:
     dự đoán same nếu cos ≥ t
     precision, recall, F0.5 (nhấn mạnh precision)
Chọn t nhỏ nhất có precision ≥ 0.97 trên cả EN và VI.
Xuất: eval/results.csv + biểu đồ precision–recall cho blog.
```

Chi phí embed 4.000 câu rất nhỏ; chạy qua chính gateway để có usage.

## 5. Workload phát lại (đo tiết kiệm)

- `eval/workload.jsonl`: 5.000 request từ ~400 câu FAQ gốc, mỗi câu có 3–8 cách diễn đạt, phân bố Zipf.
- Chạy với mock provider có giá giả định (đặt bằng giá thật của model tham chiếu) ⇒ không tốn tiền nhưng chi phí tính đúng công thức.
- 3 cấu hình: `no-cache`, `exact`, `exact+semantic` ⇒ tổng USD, hit-rate, p50/p99 latency.
- Lấy mẫu 100 câu trả lời semantic hit, chấm thủ công "trả lời đúng câu hỏi không" ⇒ precision thực tế.

## 6. Load (mock provider)

| Kịch bản | Cấu hình | Ngưỡng |
|---|---|---|
| L1 overhead | mock 0 ms, non-stream, 2k rps | p99 gateway − p99 mock < 5 ms |
| L2 exact hit | 100 % hit | p99 < 10 ms |
| L3 streaming | 1.000 VU, mock 50 token/s, 500 token | không chunk nào chậm > 100 ms so với mock; RAM < 512 MB |
| L4 hỗn hợp | 60 % miss, 30 % exact, 10 % semantic | ghi số |

## 7. Chaos

| ID | Kịch bản | Kỳ vọng |
|---|---|---|
| K1 | Mock "openai" trả 503 100 % trong 2 phút | alias `smart` thành công ≥ 99,5 % qua target 2; breaker mở trong < 30 s |
| K2 | Lỗi 30 % ngẫu nhiên | Thành công ≥ 99 %, latency tăng hợp lý |
| K3 | TTFT 15 s | Fallback sau 10 s |
| K4 | Redis chết | Request vẫn phục vụ; rate limit chuyển local; budget qua Postgres |
| K5 | Postgres chết 60 s | Usage vào spool, sau hồi phục không mất bản ghi (đếm khớp) |
| K6 | Client ngắt giữa stream | Upstream bị huỷ trong < 1 s; usage ghi phần đã sinh |

## 8. Tiêu chí release

- [ ] Compatibility test với SDK Python và Node xanh
- [ ] Test cô lập tenant xanh
- [ ] Eval có ngưỡng được chọn và lý do
- [ ] L1–L4, K1–K6 có số liệu
