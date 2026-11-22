# SPEC — llmgate v1.0

## 1. Phạm vi

**Trong:** proxy LLM tương thích OpenAI, định tuyến đa provider, cache, giới hạn, ngân sách, ghi nhận chi phí, quan sát.
**Ngoài:** fine-tuning, quản lý prompt/version, UI chat, agent tool execution, lưu hội thoại lâu dài.

## 2. Yêu cầu chức năng

| ID | Yêu cầu |
|---|---|
| F1 | `POST /v1/chat/completions` nhận đúng schema OpenAI; `model` là alias hoặc `provider:model`. |
| F2 | `stream: true` trả SSE `data: {chunk}\n\n` … `data: [DONE]`, giống OpenAI. |
| F3 | Xác thực bằng `Authorization: Bearer lg-…` (virtual key). Key không hợp lệ ⇒ 401. |
| F4 | Rate limit RPM theo key và TPM theo tenant ⇒ 429 + `Retry-After` + header RateLimit. |
| F5 | Budget theo tenant (ngày, tháng). ≥ 80 % ⇒ cảnh báo; ≥ 100 % ⇒ 402 `budget_exceeded`. Request đang chạy khi vượt không bị cắt. |
| F6 | Exact cache bật mặc định cho `temperature = 0` hoặc có `seed`; tắt được bằng header `Cache-Control: no-store`. |
| F7 | Semantic cache chỉ dùng khi đủ điều kiện §4.2 và tenant bật. |
| F8 | Routing alias theo priority hoặc weighted; fallback theo §5. |
| F9 | Mỗi request (kể cả lỗi, cache hit) có một bản ghi usage. |
| F10 | Admin API: tenant, key, budget, giá, xem usage tổng hợp, xoá cache tenant. |
| F11 | `GET /v1/models` trả alias mà key được phép dùng. |
| F12 | Key có thể bị giới hạn danh sách model/alias được phép. |

## 3. Phi chức năng

| ID | Yêu cầu |
|---|---|
| N1 | Overhead p99 < 5 ms (không cache, mock 0 ms), < 10 ms khi exact hit |
| N2 | 1.000 stream đồng thời / instance 2 vCPU, RAM < 512 MB |
| N3 | Không mất bản ghi usage khi shutdown bình thường; khi crash mất tối đa 1 s buffer (ghi rõ) |
| N4 | Không bao giờ trả cache của tenant khác (test bắt buộc) |
| N5 | Log không chứa nội dung prompt/response mặc định (`LG_LOG_BODIES=false`); key provider không xuất hiện trong log/trace |

## 4. Cache

### 4.1 Exact cache key

```
sha256( tenant_id | model_alias | canonical_json({
  messages, temperature, top_p, max_tokens, tools, tool_choice,
  response_format, seed, stop }) )
```

### 4.2 Semantic cache — điều kiện dùng

| Điều kiện | Lý do |
|---|---|
| `temperature ≤ 0.3` | Người dùng chấp nhận câu trả lời ổn định |
| Không có `tools`, `n = 1` | Kết quả tool phụ thuộc trạng thái ngoài |
| Tenant bật `semantic_cache.enabled` | Một số tenant cần cá nhân hoá |
| Câu user cuối ≤ 2.000 token | Câu dài hiếm khi lặp và embed tốn |
| Không có header `X-LLMGate-Cache: deny` | |

Khoá tra cứu: `(tenant_id, model_alias, context_hash, embedding)` với `context_hash = sha256(system_prompt + mọi message trừ user cuối)`.
Hit khi `cosine_similarity ≥ threshold` (mặc định 0,95; chọn lại theo eval, ưu tiên precision ≥ 97 %).
Phản hồi cache hit có header `X-LLMGate-Cache: semantic; similarity=0.973` và **chi phí = chi phí embedding**.

### 4.3 Vô hiệu hoá
TTL theo tenant (mặc định 7 ngày semantic, 24 h exact); `DELETE /admin/tenants/{id}/cache`.

## 5. Routing và fallback

```yaml
aliases:
  smart:
    strategy: priority
    targets:
      - { provider: anthropic, model: "<model-id>", timeout_ttft: 10s }
      - { provider: openai,    model: "<model-id>" }
  fast:
    strategy: weighted
    targets:
      - { provider: openai, model: "<model-id>", weight: 70 }
      - { provider: gemini, model: "<model-id>", weight: 30 }
  mock:
    strategy: priority
    targets: [{ provider: mock, model: echo }]
```

> Ghi model ID thật lúc cài đặt, lấy từ trang tài liệu của từng provider; không hard-code trong code.

| Lỗi upstream | Retry cùng target | Fallback target kế |
|---|---|---|
| 429 có `retry-after` ≤ 2 s | 1 lần | sau retry |
| 429 khác, 500, 502, 503, 504, lỗi kết nối | không | có |
| Timeout TTFT | không | có |
| 400, 401, 403, 404, content filter | không | **không** (lỗi của request/cấu hình) |
| Lỗi sau khi đã gửi byte đầu cho client | không | **không** — gửi chunk lỗi rồi đóng (ADR-0004) |

Circuit breaker mỗi `provider:model`: mở khi ≥ 50 % lỗi trong cửa sổ 30 s (≥ 10 request); half-open sau 15 s.

## 6. Chi phí

```
cost_usd = (input_tokens - cached_input_tokens) * price_in
         + cached_input_tokens * price_cached_in
         + output_tokens * price_out
         (giá / 1_000_000; lưu micro-USD dạng BIGINT)
```

`pricing.yaml`:
```yaml
- model: "openai:<model-id>"
  effective_from: 2026-11-01
  input_per_mtok: 0.00      # điền giá công bố lúc triển khai
  cached_input_per_mtok: 0.00
  output_per_mtok: 0.00
```

## 7. Mô hình dữ liệu (rút gọn)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE tenants (
  id UUID PRIMARY KEY, name TEXT NOT NULL,
  monthly_budget_micro_usd BIGINT, daily_budget_micro_usd BIGINT,
  rpm_limit INT, tpm_limit INT,
  semantic_cache_enabled BOOLEAN DEFAULT false,
  semantic_threshold REAL DEFAULT 0.95,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE virtual_keys (
  id UUID PRIMARY KEY, tenant_id UUID REFERENCES tenants(id),
  key_hash BYTEA UNIQUE NOT NULL, key_prefix TEXT NOT NULL,  -- "lg-abc1"
  allowed_aliases TEXT[], rpm_limit INT,
  revoked_at TIMESTAMPTZ, created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE provider_credentials (
  provider TEXT PRIMARY KEY, ciphertext BYTEA NOT NULL, nonce BYTEA NOT NULL, key_version INT NOT NULL
);

CREATE TABLE usage_records (
  id BIGSERIAL, request_id UUID NOT NULL, tenant_id UUID NOT NULL, key_id UUID,
  alias TEXT, provider TEXT, model TEXT, endpoint TEXT,
  status INT, error_code TEXT, cache TEXT,            -- none|exact|semantic
  input_tokens INT, cached_input_tokens INT, output_tokens INT,
  cost_micro_usd BIGINT NOT NULL, saved_micro_usd BIGINT DEFAULT 0,
  ttft_ms INT, latency_ms INT, attempts INT, fallback_from TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE UNIQUE INDEX ON usage_records (request_id, created_at);

CREATE TABLE semantic_cache (
  id BIGSERIAL PRIMARY KEY, tenant_id UUID NOT NULL, model_alias TEXT NOT NULL,
  context_hash BYTEA NOT NULL, embedding vector(1536) NOT NULL,
  prompt_preview TEXT, response JSONB NOT NULL,
  hits INT DEFAULT 0, created_at TIMESTAMPTZ DEFAULT now(), expires_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX ON semantic_cache USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON semantic_cache (tenant_id, model_alias, context_hash);
```

`saved_micro_usd` = chi phí ước tính nếu không có cache ⇒ dashboard "đã tiết kiệm".

## 8. Mã lỗi (format OpenAI)

| HTTP | `error.code` |
|---|---|
| 400 | `invalid_request` |
| 401 | `invalid_api_key` |
| 403 | `model_not_allowed` |
| 402 | `budget_exceeded` |
| 429 | `rate_limit_exceeded` (`type: requests` hoặc `tokens`) |
| 502 | `upstream_error` (mọi target đều lỗi; `message` liệt kê đã thử gì) |
| 504 | `upstream_timeout` |

## 9. Cấu hình (env)

| Biến | Mặc định |
|---|---|
| `LG_HTTP_ADDR` | `:8000` |
| `LG_DATABASE_URL`, `LG_REDIS_ADDR` | |
| `LG_ROUTES_FILE`, `LG_PRICING_FILE` | `/etc/llmgate/*.yaml` |
| `LG_MASTER_KEY_FILE` | khoá AES-256 để giải mã credential |
| `LG_EMBED_ALIAS` | `embed` |
| `LG_USAGE_FLUSH_INTERVAL` / `_BATCH` | `1s` / `500` |
| `LG_LOG_BODIES` | `false` |
| `LG_ADMIN_TOKEN_FILE` | |
