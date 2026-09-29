# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Báo cáo cá nhân của Nguyễn Thị Minh Khánh (MSSV: 2A202602546) - Lớp K4-L3A.

## 1. Thông tin học viên

- **Họ và tên:** Nguyễn Thị Minh Khánh
- **MSSV:** 2A202602546
- **Lớp:** K4-L3A
- **Repository URL:** `https://github.com/nguynkhanh04/K4-L3A-DAY13-NguyenThiMinhKhanh-2A202602546-Monitoring-LLMOps`
- **Commit SHA cuối:** `7857e56d4c57a1185925a31453a4cb5b17c798ba`
- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602546`

## 2. Evidence index

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đạt 100/100 sau khi bind contextvars & scrub PII |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Đạt contract 6/6 panel đầy đủ |
| `pytest` | 22 passed | 22 passed | Pass 100% test suite (22/22 tests) |
| Số traces hợp lệ | 10+ | 10+ | Đã tạo traces với span tree đầy đủ trên Langfuse |
| Số PII leak | 0 | 0 | Scrub toàn bộ PII (email, phone, cccd, credit card) |
| Latency P95 / TTFT P95 | ~150ms / ~50ms | ~150ms / ~50ms | Latency ổn định trong môi trường mock |
| Retrieval success rate | 100% | 100% | Retrieval hoạt động bình thường |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware` (`app/middleware.py`), middleware kiểm tra header `x-request-id` hoặc `x-correlation-id` từ request tới; nếu chưa có thì sinh ID dạng `req-<8-hex>` (`req-{uuid.uuid4().hex[:8]}`). Gọi `clear_contextvars()` ở đầu mỗi request để xóa context cũ, bind `correlation_id` vào `structlog.contextvars`, gán vào `request.state.correlation_id` và đính kèm vào response header (`x-request-id`, `x-correlation-id`, `x-response-time-ms`).
- **Các metadata được ghi vào structured log:** `ts`, `level`, `service`, `event`, `correlation_id`, `user_id_hash` (mã hóa SHA-256 12 ký tự), `session_id`, `feature`, `model`, `env`, cùng các thông số performance (`latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success`, `payload`).
- **Cách bảo đảm PII được scrub trước khi ghi:** Đăng ký processor `scrub_event` trong `structlog.configure()` trước bước JSON renderer và `JsonlFileProcessor`. Processor duyệt qua toàn bộ giá trị string trong log dict (bao gồm `payload` và `event`) và áp dụng `scrub_text` trong `app/pii.py` để thay thế email, số điện thoại Việt Nam, CCCD 12 số, thẻ credit card thành thẻ `[REDACTED_*]`.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` thu được kết quả `100/100` không chứa PII thô và 100% log chứa đầy đủ required & enrichment fields.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Cấu hình `LANGFUSE_PUBLIC_KEY` và `LANGFUSE_SECRET_KEY` thuộc project `day13-k4-l3a-2A202602546` trong file `.env`.
- **Cấu trúc root/retrieval/generation observations:** Root observation `@observe(name="lab-agent-run", as_type="agent")` cho `LabAgent.run`, child observation `@observe(name="retrieval", as_type="retriever")` cho `_retrieve()`, và child observation `@observe(name="llm-generation", as_type="generation")` cho `_generate()`. Span `generation` chứa model, input/output text, token counts (`input_tokens`, `output_tokens`), `total_cost` và prompt object.
- **Cách nối trace với log:** Ghi `correlation_id` vào trace metadata trong `propagate_attributes(metadata={"correlation_id": correlation_id, ...})`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** `v1` (label: `production`)
- **Version/label candidate:** `v2` (label: `staging`)
- **Trace ID của mỗi version:** Thu thập từ Langfuse dashboard khi chạy workload với v1/v2.
- **Cách promote và rollback `production`:** Trong Langfuse Prompt Management UI, chuyển label `production` sang prompt version mới (promote) hoặc gán lại label `production` cho version cũ (rollback). Hệ thống tự động fetch prompt theo label `production`.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Panel Latency (P50/P95/P99, TTFT), Traffic (rate/min), Errors (Error rate, Retrieval success rate), Cost (Cost over time), Tokens (Input/Output sum), Quality (Quality score mean).
- **SLO và lý do chọn:** SLO `fast_successful_requests`: 99.5% request hoàn thành trong window 28d với latency <= 3000ms.
- **Cách tính error budget:** 0.5% tổng số request nhận được trong 28d được phép có latency > 3000ms hoặc bị lỗi.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95`: Warning khi P95 > 3000ms kéo dài 5 phút (Runbook `docs/alerts.md#alert-1`).
  2. `HighErrorRate`: Critical khi Error Rate > 2% kéo dài 3 phút (Runbook `docs/alerts.md#alert-2`).
  3. `RetrievalFailureRate`: Warning khi Retrieval success < 90% kéo dài 5 phút (Runbook `docs/alerts.md#alert-3`).

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1` (Incident scenario: `rag_slow`, affected feature: `monitoring`, seed: `1311`, latency threshold: `2000ms`).
- **Khoảng thời gian điều tra:** `2026-09-29T09:07:13.132Z` – `2026-09-29T09:07:36.286Z` (UTC).
- **Triệu chứng từ metrics:**
  - `latency_p50`: tăng vọt lên **2919.0ms** (so với baseline bình thường ~150ms).
  - `latency_p95`: tăng vọt lên **3778.0ms** (vượt xa latency threshold 2000ms của challenge và ngưỡng SLO 3000ms).
  - `ttft_p95`: duy trì ở mức thấp **56.0ms** (cho thấy bước generate LLM bắt đầu rất nhanh sau khi nhận prompt, độ trễ nằm ở giai đoạn trước khi gọi LLM).
  - `error_breakdown`: không có lỗi HTTP/500 (tất cả 5 requests trong challenge đều trả về HTTP 200).
- **Log line và correlation ID liên quan:**
  - Request tiêu biểu: `correlation_id`: `req-39959b72` (session: `k4-l3a-challenge-s04`, user_hash: `4570299f37e2`, feature: `monitoring`):
    - Log nhận request (`request_received` lúc `2026-09-29T09:07:19.728987Z`):
      `{"service": "api", "payload": {"message_preview": "Which signal should be checked after latency increases?"}, "event": "request_received", "feature": "monitoring", "session_id": "k4-l3a-challenge-s04", "model": "claude-sonnet-4-5", "user_id_hash": "4570299f37e2", "env": "dev", "correlation_id": "req-39959b72", "level": "info", "ts": "2026-09-29T09:07:19.728987Z"}`
    - Log trả response (`response_sent` lúc `2026-09-29T09:07:24.594348Z`):
      `{"service": "api", "latency_ms": 3778, "ttft_ms": 50, "tokens_in": 36, "tokens_out": 144, "cost_usd": 0.002268, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "payload": {"answer_preview": "Starter answer. You should improve this output logic and add better quality chec..."}, "event": "response_sent", "feature": "monitoring", "session_id": "k4-l3a-challenge-s04", "model": "claude-sonnet-4-5", "user_id_hash": "4570299f37e2", "env": "dev", "correlation_id": "req-39959b72", "level": "info", "ts": "2026-09-29T09:07:24.594348Z"}`
  - Các correlation IDs khác bị ảnh hưởng: `req-76f34343` (latency 2910ms), `req-26bf46ab` (latency 2919ms), `req-eeffebc8` (latency 2919ms), `req-95f98c4c` (latency 2922ms).
- **Trace ID và span gây ảnh hưởng:**
  - Langfuse Trace ID tương ứng với `req-39959b72`: `6ee4881cef348d485edb0dde38294d44` (Project: `day13-k4-l3a-2A202602546`).
  - Phân tích span tree trong Trace:
    - Root observation `lab-agent-run` (Observation ID: `7f8ed6233195149a`, Type: `AGENT`): tổng thời gian thực thi là **3.779s**.
    - Child span `retrieval` (Observation ID: `bf2e2e4db5de7e50`, Type: `RETRIEVER`): thời gian thực thi là **2.502s** (chiếm ~66% tổng latency và là nguyên nhân trực tiếp gây nghẽn).
    - Child span `llm-generation` (Observation ID: `d815003e3d587d69`, Type: `GENERATION`): thời gian thực thi chỉ **0.151s** (hoạt động bình thường).
- **Root cause:**
  - Module retrieval (`app/mock_rag.py`) gặp sự cố suy giảm hiệu năng do cờ incident `rag_slow` được bật (mô phỏng tình trạng Vector Database/Corpus search bị nghẽn I/O dẫn đến blocking delay 2.5s trong hàm `retrieve()`).
- **Fix action:**
  - Ngay lập tức vô hiệu hóa cờ sự cố bằng lệnh `python scripts/inject_incident.py --disable` hoặc gọi API `POST /incidents/rag_slow/disable`.
  - Trong môi trường production thật: kiểm tra tình trạng kết nối tới Vector DB cluster, scale read replicas cho retriever service, hoặc kích hoạt fallback mechanism (circuit breaker) để trả về fallback document khi retrieval latency vượt quá ngưỡng an toàn.
- **Preventive measure:**
  - Thiết lập timeout cho retrieval client (ví dụ 1.5s max) kèm cơ chế circuit breaker và fallback sang prompt không dùng context nếu Vector DB bị timeout.
  - Cấu hình cảnh báo sớm `HighLatencyP95` (P95 > 3000ms trong 5 phút) để đội trực vận hành phát hiện ngay khi có suy giảm hiệu năng trước khi ảnh hưởng diện rộng.
  - Tách riêng panel và metric giám sát chuyên biệt cho retrieval latency (`retrieval_latency_ms`) song song với LLM latency.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đặt `scrub_event` processor làm bước tiền xử lý ngay trong chuỗi processor của `structlog` trước khi ghi file `JsonlFileProcessor`. Quyết định này giúp đảm bảo dữ liệu PII được làm sạch ở mức tập trung, không bị bỏ sót dù log được gọi ở bất kỳ module nào.
- **Một lỗi/blocker đã gặp:** Trong lượt chạy baseline đầu tiên, `validate_logs.py` báo score 30/100 do các dòng log cũ đã tồn tại trước khi sửa code.
- **Cách tìm nguyên nhân và xử lý:** Đọc kỹ `docs/GUIDE.md`, xóa file log cũ `data/logs.jsonl`, khởi động lại workload và chạy lại `validate_logs.py` đạt 100/100.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  1. Metrics cảnh báo hệ thống có triệu chứng bất thường (ví dụ latency tăng đột biến hoặc error rate vượt ngưỡng).
  2. Logs giúp tra cứu chi tiết request cụ thể bị ảnh hưởng dựa trên `correlation_id` và timestamp.
  3. Traces (Langfuse) giúp đào sâu vào span tree của request đó để khoanh vùng chính xác component gây ra lỗi/chậm (ví dụ span retrieval hay generation).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt version giúp quản lý vòng đời prompt chủ động và rollback an toàn khi prompt mới có chất lượng kém; theo dõi token/cost giúp tránh bùng nổ chi phí API; SLO bảo đảm chất lượng dịch vụ cam kết với người dùng.
- **Điều quan trọng nhất đã học:** Cách thiết lập observability toàn diện cho hệ thống LLMOps kết hợp giữa Structured Logging, Distributed Tracing và Metric Monitoring.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Không có.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
