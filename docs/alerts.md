# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1

- Tên: HighLatencyP95
- Severity: Warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-warning)
- SLI/SLO liên quan: `fast_successful_requests` (Latency P95 <= 3000ms)
- Điều kiện và thời gian duy trì: `p95_latency_ms > 3000` trong 5 phút liên tục.
- Ảnh hưởng tới người dùng: Người dùng phản hồi hệ thống phản hồi chậm, trải nghiệm chat bị trì hoãn.
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard quan sát panel Latency (P95/P99) và TTFT để xác định triệu chứng.
  2. Lọc `data/logs.jsonl` lấy correlation_id của các request có `latency_ms > 3000`.
  3. Tra cứu trace ID trên Langfuse để xem waterfall span nào chậm (retrieval hay generation).
- Mitigation tạm thời: Scale up worker nodes hoặc switch prompt / retrieval fallback nhanh.
- Owner: devops-team

## Alert 2

- Tên: HighErrorRate
- Severity: Critical
- Duration: 3m
- Kênh thông báo: Slack (#alerts-critical)
- SLI/SLO liên quan: Error Rate Guardrail (<= 2%)
- Điều kiện và thời gian duy trì: `error_rate_pct > 2` trong 3 phút liên tục.
- Ảnh hưởng tới người dùng: Request của người dùng bị từ chối hoặc trả lỗi 500 liên tục.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Error rate trên Dashboard để xem tỷ lệ lỗi hiện tại.
  2. Lọc `data/logs.jsonl` tìm log event `request_failed` và xem field `error_type`.
  3. Tra cứu trace ID tương ứng trên Langfuse để kiểm tra exception chi tiết tại span bị đứt.
- Mitigation tạm thời: Bật circuit breaker hoặc fallback service để trả câu trả lời mặc định an toàn.
- Owner: backend-team

## Alert 3

- Tên: RetrievalFailureRate
- Severity: Warning
- Duration: 5m
- Kênh thông báo: Slack (#alerts-warning)
- SLI/SLO liên quan: Retrieval Success Rate Guardrail (>= 90%)
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90` trong 5 phút liên tục.
- Ảnh hưởng tới người dùng: Mô hình không truy xuất được tri thức domain, chất lượng câu trả lời bị suy giảm.
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Errors để xem tỉ lệ `tool_success_rate_pct` của tool `retrieval`.
  2. Lọc log tìm các request có `tool_success == false` hoặc `error_type == RuntimeError`.
  3. Mở Langfuse trace tìm span `retrieval` để xác định lỗi timeout hoặc rớt kết nối vector db.
- Mitigation tạm thời: Chuyển sang cache retrieval hoặc fallback corpus đơn giản.
- Owner: ai-team

