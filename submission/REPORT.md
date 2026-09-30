# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đỗ Phúc Hưng
- **MSSV:** 2A202602762
- **Lớp:** K4-L3A
- **Repository URL:**
- **Commit SHA cuối:**
- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602762`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

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
| `validate_logs.py` | 30/100 | 100/100 | Đạt 100/100 điểm, đầy đủ correlation ID, enrichment và PII scrubbed |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Đã đạt hợp lệ contract 6/6 panel |
| `pytest` | 22/22 passed | 22/22 passed | Đạt 22/22 unit tests |
| Số traces hợp lệ | | | |
| Số PII leak | 0 | 0 | Đã kiểm tra đệ quy qua processor `scrub_event` không lộ PII |
| Latency P95 / TTFT P95 | | | |
| Retrieval success rate | | | |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Trong `CorrelationIdMiddleware`, kiểm tra header `x-request-id` từ request. Nếu có thì sử dụng, nếu chưa có thì tự động sinh mới theo định dạng `req-<8-hex>` (`req-` + `uuid.uuid4().hex[:8]`). Trước mỗi request, gọi `clear_contextvars()` để tránh rò rỉ context giữa các request và dùng `bind_contextvars(correlation_id=...)` để gắn vào structlog. Đồng thời gán correlation ID vào `request.state` và trả lại header `x-request-id` cùng `x-response-time-ms` trong HTTP response.
- **Các metadata được ghi vào structured log:** `ts` (ISO timestamp), `level`, `service="api"`, `event`, `correlation_id`, `user_id_hash` (hashing SHA-256 12 ký tự), `session_id`, `feature`, `model`, `env`, cùng các thông số latency (`latency_ms`, `ttft_ms`), token (`tokens_in`, `tokens_out`), `cost_usd`, `quality_score`, `payload`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Xây dựng processor `scrub_event` và đăng ký trong chuỗi structlog processors trước `JsonlFileProcessor()` và `JSONRenderer()`. Processor duyệt đệ quy toàn bộ giá trị string trong log dict và áp dụng Regex thay thế PII (`email`, `phone_vn`, `cccd`, `credit_card`) thành các tag `[REDACTED_*]`.
- **Cách kiểm chứng kết quả:** Xóa file log cũ (`data/logs.jsonl`), chạy lại `python scripts/load_test.py` và gọi `python scripts/validate_logs.py` thu được kết quả **100/100** với 0 rò rỉ PII và 10 unique correlation IDs hợp lệ.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
- **Cấu trúc root/retrieval/generation observations:**
- **Cách nối trace với log:**
- **Prompt name:**
- **Version/label baseline:**
- **Version/label candidate:**
- **Trace ID của mỗi version:**
- **Cách promote và rollback `production`:**

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
- **SLO và lý do chọn:**
- **Cách tính error budget:**
- **Ba alert và runbook tương ứng:**

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3a-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** `2026-09-30T02:15:11Z` đến `2026-09-30T02:15:34Z` (múi giờ UTC / `09:15:11` - `09:15:34` UTC+7)
- **Triệu chứng từ metrics:** Latency trung bình và P95/P99 tăng vọt vượt ngưỡng quy định `latency_threshold_ms` (2000ms). Latency của các request thuộc tính năng `monitoring` tăng đột biến từ mức bình thường ~150ms lên **2653ms - 5402ms**. Chỉ số TTFT (Time To First Token) vẫn duy trì ổn định ở mức ~50ms, cho thấy nguyên nhân chậm không phải do LLM generation mà xuất phát từ bước tiền xử lý/retrieval.
- **Log line và correlation ID liên quan:**
  - Control log chèn incident: `{"service": "control", "payload": {"name": "rag_slow"}, "event": "incident_enabled", "correlation_id": "req-4a83a642", "level": "warning", "ts": "2026-09-30T02:15:11.290934Z"}`
  - Structured log đại diện cho request bị ảnh hưởng (Correlation ID: `req-af3a60aa`, User Hash: `4570299f37e2`, Session: `k4-l3a-challenge-s04`):
    `{"service": "api", "latency_ms": 5172, "ttft_ms": 50, "tokens_in": 36, "tokens_out": 139, "cost_usd": 0.002193, "quality_score": 0.9, "tool_name": "retrieval", "tool_success": true, "payload": {"answer_preview": "Starter answer..."}, "event": "response_sent", "correlation_id": "req-af3a60aa", "user_id_hash": "4570299f37e2", "model": "claude-sonnet-4-5", "session_id": "k4-l3a-challenge-s04", "feature": "monitoring", "env": "dev", "level": "info", "ts": "2026-09-30T02:15:20.209640Z"}`
- **Trace ID và span gây ảnh hưởng:**
  - Trace tương ứng chứa metadata `correlation_id: req-af3a60aa` trong project Langfuse cá nhân.
  - Span bị tắc nghẽn: child observation `retrieval` (gọi hàm `retrieve()` trong `app/mock_rag.py`). Span này chiếm trọn 2500ms+ delay (trên 95% tổng thời gian phản hồi của request).
- **Root cause:** Incident `rag_slow` đã bị kích hoạt trên hệ thống (`STATE["rag_slow"] = True`), chèn trực tiếp `time.sleep(2.5)` vào bước `retrieve()` vector store trong `app/mock_rag.py`.
- **Fix action:**
  - Tắt incident `rag_slow` bằng lệnh `python scripts/inject_incident.py --disable` (hoặc gửi request `POST /incidents/rag_slow/disable`).
  - Thực hiện load test kiểm thử lại: latency hạ xuống mức an toàn < 200ms và đạt SLO đề ra.
- **Preventive measure:**
  - Cấu hình Alert Rule với điều kiện P95 Latency > 2000ms kéo dài trong 1 phút, gửi cảnh báo qua Slack kèm Runbook xử lý.
  - Thiết lập timeout tối đa (vd: 1.0s) cho tác vụ retrieval kèm cơ chế fallback tự động (fallback answer/cache response) khi vector store phản hồi chậm.
  - Thường xuyên giám sát span waterfall trên Langfuse để phát hiện sớm bottlenecks ở các dịch vụ hạ tầng phụ trợ.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
- **Một lỗi/blocker đã gặp:**
- **Cách tìm nguyên nhân và xử lý:**
- **Cách hiểu luồng Metrics → Logs → Traces:**
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
- **Điều quan trọng nhất đã học:**
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
