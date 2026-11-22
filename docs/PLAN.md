# PLAN — llmgate

Thời lượng: **4 tuần** (~55 giờ). Ngân sách gọi API thật cho toàn dự án: **≤ 5 USD** (đặt hard limit ngay trên chính gateway).

## Phase 0 — Khung (0,5 ngày)

- [ ] `go mod init github.com/dvphuc96/llmgate`; import `github.com/dvphuc96/ratelimitd/pkg/limiter`
- [ ] Compose: postgres:16 + pgvector, redis:7, llmgate, mock-provider, prometheus, grafana, jaeger
- [ ] `.env.example` với `OPENAI_API_KEY=`, `ANTHROPIC_API_KEY=`, `GEMINI_API_KEY=` (để trống ⇒ chỉ mock)

## Phase 1 — Proxy tương thích OpenAI + adapter + streaming (tuần 1, ~14 giờ)

- [ ] Kiểu dữ liệu chuẩn nội bộ `ChatRequest`/`ChatResponse`/`ChatChunk` (theo schema OpenAI Chat Completions)
- [ ] Interface:
  ```go
  type Provider interface {
      Name() string
      Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
      ChatStream(ctx context.Context, req *ChatRequest) (ChunkStream, error)
      Embed(ctx context.Context, req *EmbedRequest) (*EmbedResponse, error)
  }
  ```
- [ ] Adapter OpenAI (gần như passthrough), Anthropic (chuyển `system`, `messages`, `max_tokens` bắt buộc, map stop_reason, SSE event types), Gemini (`contents/parts`, `generationConfig`)
- [ ] Mock provider (binary riêng + in-process): env `MOCK_LATENCY_MS`, `MOCK_TTFT_MS`, `MOCK_TOKENS_PER_S`, `MOCK_ERROR_RATE`, `MOCK_STATUS=429|500|timeout`
- [ ] Streaming SSE: chuyển chunk ngay (flush mỗi chunk), hỗ trợ client ngắt kết nối (huỷ context upstream), heartbeat comment `: ping` mỗi 15 s
- [ ] Bắt `usage` từ provider (OpenAI `stream_options.include_usage`, Anthropic `message_delta`); thiếu thì ước lượng bằng tokenizer (`tiktoken-go`)
- [ ] Lỗi chuẩn hoá theo format lỗi OpenAI (`{error:{type,code,message}}`)
- [ ] Test tương thích: chạy chính OpenAI SDK (Python `openai`) trỏ `base_url` vào gateway, cả stream và non-stream

**Done khi:** script Python dùng SDK OpenAI chat được với 4 provider qua cùng một endpoint, stream hiển thị từng token.

## Phase 2 — Virtual key, rate limit, budget, usage ledger (tuần 2, ~14 giờ)

- [ ] Bảng `tenants`, `virtual_keys` (lưu SHA-256 của key, prefix `lg-`), `provider_credentials` (AES-GCM, khoá chủ từ env/file)
- [ ] Admin API: tạo tenant, tạo/thu hồi key, đặt limit và budget, xem usage
- [ ] Rate limit 2 tầng dùng `ratelimitd`:
  - RPM theo key (hits = 1)
  - TPM theo tenant: **trước** khi gọi trừ `estimated = prompt_tokens + max_tokens`; **sau** khi xong điều chỉnh theo thực tế (hoàn lại phần dư bằng hits âm trên token bucket — xem ADR-0003)
- [ ] Bảng giá `pricing.yaml` (USD / 1M token input, output, cached input) theo model; có `effective_from` để đổi giá không làm sai lịch sử
- [ ] Budget: bộ đếm Redis `spend:{tenant}:{yyyymm}` (micro-USD, INCRBY) để kiểm nhanh; Postgres là sổ chính; đối chiếu mỗi 5 phút
- [ ] Soft limit 80 % ⇒ header `X-LLMGate-Budget-Warning`, webhook cảnh báo; hard limit 100 % ⇒ 402 `budget_exceeded`
- [ ] Usage ledger: ghi bất đồng bộ qua channel có buffer + batch `COPY` mỗi 1 s hoặc 500 dòng; khi buffer đầy ⇒ ghi đồng bộ (không bao giờ mất bản ghi tính tiền)
- [ ] Header phản hồi: `X-LLMGate-Request-Id`, `X-LLMGate-Cache`, `X-LLMGate-Provider`, `X-LLMGate-Cost-USD`

**Done khi:** tenant có budget 0,01 USD bị chặn đúng lúc; tổng `usage` khớp bộ đếm Redis sai số < 0,1 %.

## Phase 3 — Cache và routing thông minh (tuần 3, ~14 giờ)

- [ ] Chuẩn hoá request để băm: sắp xếp key JSON, bỏ trường không ảnh hưởng (`user`, `stream`), giữ `model`, `messages`, `temperature`, `tools`, `response_format`, `seed`
- [ ] Exact cache Redis: `cache:exact:{tenant}:{sha256}` TTL mặc định 24 h; cache cả kết quả stream (lưu chunk, phát lại với tốc độ nhanh)
- [ ] Semantic cache:
  - [ ] Điều kiện được dùng: `temperature ≤ 0.3`, không có `tools`, không `n > 1`, header `X-LLMGate-Cache: allow` hoặc cấu hình tenant bật
  - [ ] Embed **tin nhắn user cuối + hash của system prompt và lịch sử trước đó** (ADR-0002) — câu giống nhau nhưng ngữ cảnh khác không được trùng
  - [ ] pgvector `HNSW (vector_cosine_ops)`, lọc `tenant_id`, `model_alias`, `context_hash`; top-1; hit khi `similarity ≥ threshold` (mặc định 0,95, chỉnh theo eval)
  - [ ] Embedding model qua chính gateway (mock: embedding xác định từ hash để test)
  - [ ] TTL + xoá theo tenant; endpoint invalidate
- [ ] Router:
  - [ ] Alias: `smart → [anthropic:<model>, openai:<model>]`, `fast → weighted 70/30`
  - [ ] Fallback khi 429, 5xx, timeout kết nối, timeout TTFT (mặc định 10 s); **không** fallback khi đã stream byte đầu tiên cho client (ADR-0004)
  - [ ] Circuit breaker theo provider+model; health score từ tỉ lệ lỗi + latency EWMA
  - [ ] Retry cùng provider tối đa 1 lần với backoff + jitter cho 429 có `retry-after` ngắn
- [ ] Metric: `llm_requests_total{provider,model,status,cache}`, `llm_tokens_total{direction}`, `llm_cost_usd_total{tenant,model}`, `llm_ttft_seconds`, `cache_similarity` histogram

**Done khi:** eval semantic cache có đường cong precision/hit-rate; tắt OpenAI giả lập ⇒ request alias `smart` vẫn thành công qua provider khác.

## Phase 4 — Đo, eval, tài liệu (tuần 4, ~13 giờ)

- [ ] Bộ eval semantic cache (`eval/`): ~2.000 cặp câu có nhãn "cùng ý / khác ý" (Quora Question Pairs mẫu nhỏ + 300 cặp tiếng Việt tự viết theo chủ đề CSKH); quét ngưỡng 0,80 → 0,99; xuất precision, recall, hit-rate
- [ ] Workload phát lại: 5.000 câu hỏi FAQ có phân bố lặp thực tế (Zipf), chạy 3 cấu hình: không cache / exact / exact+semantic ⇒ chi phí USD, latency
- [ ] Load test k6 với mock provider: overhead gateway, streaming 1.000 kết nối đồng thời
- [ ] Chaos: provider chính lỗi 100 %, 30 %, chậm TTFT 15 s; Redis chết (cache bỏ qua, rate limit theo chế độ lỗi của ratelimitd, budget dùng Postgres)
- [ ] Dashboard Grafana: chi phí theo tenant/model/ngày, hit-rate cache, tiết kiệm ước tính, lỗi theo provider
- [ ] Báo cáo "chi phí trên mỗi khách hàng": tenant trả 20 USD/tháng tiêu bao nhiêu USD LLM
- [ ] README + blog #3; tag `v1.0.0`

## Cắt nếu trễ

1. Gemini adapter (giữ OpenAI, Anthropic, Mock)
2. Cache kết quả stream (chỉ cache non-stream)
3. Weighted routing (giữ priority + fallback)
4. Mã hoá credential (dùng env)

## Mở rộng sau v1

- Guardrails: phát hiện PII trong prompt trước khi gửi provider, chặn prompt injection cơ bản (liên quan bài #6).
- Prompt caching phía provider (Anthropic/OpenAI cached input) và tính giá đúng.
- Smart routing theo độ khó câu hỏi (classifier nhỏ chọn `cheap` vs `smart`).
