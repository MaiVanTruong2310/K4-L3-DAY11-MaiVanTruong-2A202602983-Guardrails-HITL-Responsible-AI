# Báo Cáo Triển Khai Pha 4: Ghép Pipeline & Sinh Artifacts (Checkpoint 3)

## 1. Mục Tiêu Pha 4
Ghép nối toàn diện các lớp phòng thủ đa tầng (*Defense-in-Depth*) cho hệ thống VinBank Assistant, kích hoạt bộ kiểm thử tự động và xuất đầy đủ các artifacts bắt buộc theo đúng chuẩn schema vào thư mục `outputs/`:
- **Rate Limiting**: Ngăn chặn tấn công brute-force / spam / flooding dựa trên thuật toán sliding-window per user.
- **Audit Logging**: Ghi nhận toàn bộ nhật ký tương tác phục vụ điều tra (forensics) và kiểm toán.
- **Monitoring & Alerts**: Giám sát các chỉ số vận hành (block rate, rate limit hits, fail rate) và tự động kích hoạt cảnh báo khi vượt ngưỡng.
- **Pipeline Ordering & Egress Security**: Đảm bảo thứ tự thực thi nghiêm ngặt (`RateLimit` $\rightarrow$ `InputGuardrail` $\rightarrow$ `OutputGuardrail`) và kiểm soát chính sách xuất dữ liệu ra ngoài (*Egress Allowlist*).
- **Artifacts Generation**: Tự động sinh `outputs/results.json`, `outputs/audit_log.json`, `outputs/metrics.json`.

---

## 2. Kế Hoạch & Cách Thức Thực Hiện

### 2.1. Triển khai Rate Limiter (`src/assignment/rate_limiter.py`)
- Cài đặt `RateLimitPlugin` kế thừa từ `base_plugin.BasePlugin`:
  - Quản lý cửa sổ trượt thời gian bằng `collections.deque` theo từng `user_id`.
  - Mặc định: tối đa `max_requests = 10` trong `window_seconds = 60`.
  - Khi có request mới: loại bỏ các timestamp quá hạn `(now - window_seconds)`.
  - Nếu số lượng trong cửa sổ đạt ngưỡng: tăng `blocked_count` và trả về thông báo lỗi kèm thời gian cần chờ; nếu chưa vượt ngưỡng: lưu timestamp hiện tại và trả về `None` (cho qua).

### 2.2. Triển khai Audit Log (`src/assignment/audit_log.py`)
- Cài đặt `AuditLogPlugin`:
  - `record_input`: Ghi nhận `user_id`, nội dung truy vấn, thời gian bắt đầu (`start_time`), `timestamp` chuẩn ISO UTC.
  - `record_output`: Tính toán độ trễ xử lý (`latency_ms`), trạng thái `blocked`, tầng phòng thủ bắt giữ (`layer`), câu trả lời cuối cùng và lưu vào danh sách `logs`.
  - `export_json`: Xuất toàn bộ danh sách bản ghi ra file `outputs/audit_log.json` dưới định dạng JSON có format thụt dòng rõ ràng.

### 2.3. Triển khai Monitoring & Alerts (`src/assignment/monitoring.py`)
- Cài đặt `MonitoringAlert`:
  - Bộ đếm tập trung: `total_requests`, `blocked_requests`, `rate_limit_hits`, `judge_checks`, `judge_fails`.
  - `check_metrics`: Tự động tính toán tỷ lệ chặn (`block_rate`), tỷ lệ thẩm định lỗi (`judge_fail_rate`) và kích hoạt đối tượng `Alert` khi vượt ngưỡng cảnh báo (ví dụ `rate_limit_hit_threshold = 5`).
  - `export_json`: Xuất trạng thái snapshot và danh sách cảnh báo ra file `outputs/metrics.json`.

### 2.4. Lắp ráp Pipeline & Kiểm soát Egress (`src/assignment/pipeline.py`)
1. **Kiểm soát Egress (`is_egress_allowed`)**:
   - Phân tích cú pháp URL đích: Bắt buộc giao thức `https` và hostname phải thuộc domain cho phép của ngân hàng (`api.vinbank.example` hoặc `*.vinbank.example`). Chặn toàn bộ các domain lạ hoặc giả mạo (ví dụ `api.vinbank.example.evil.com`).
   - Phân tích payload: Quét regex để chặn hoàn toàn dữ liệu nhạy cảm (SĐT, Email, API key, DB host, mật khẩu `admin123` và các demo secrets).
2. **Thứ tự Plugin chuẩn (`build_production_plugins`)**:
   - Khởi tạo đúng thứ tự:
     1. `RateLimitPlugin` (chặn trước khi tốn tài nguyên xử lý ngôn ngữ)
     2. `InputGuardrailPlugin` (chặn injection và lạc đề trước khi gọi LLM)
     3. `OutputGuardrailPlugin` (lọc PII và bảo vệ dữ liệu sau LLM)
3. **Observability (`build_observability`)**:
   - Trả về cặp `(AuditLogPlugin(), MonitoringAlert())`.
4. **Bộ kiểm thử tự động (`run_assignment_suite`)**:
   - Thực thi tuần tự 4 nhóm kịch bản:
     - **Safe Queries** (6 câu): Tất cả đều đạt `blocked: False`.
     - **Attack Queries** (8 câu): Bị chặn bởi `input_guardrail` với `blocked: True`.
     - **Rate Limit Test** (15 requests liên tiếp): 10 passed, 5 blocked (`sent = 15`, `passed = 10`, `blocked = 5`).
     - **Edge Cases** (4 trường hợp): Chuỗi rỗng, khoảng trắng, câu hỏi ngoài ngành, và câu hỏi hợp lệ tóm tắt văn bản ngoài.
   - Ghi file `outputs/results.json` theo đúng schema chuẩn và xuất đồng thời `audit_log.json`, `metrics.json`.

---

## 3. Kết Quả Kiểm Thử (Verification)

### 3.1. Chạy Runner Checkpoint 3 (`main.py --part 3`)
```powershell
.\.venv\Scripts\python.exe src/main.py --part 3
```

**Đầu ra Terminal:**
```text
Blue  — openrouter:liquid/lfm-2.5-2.6b  [LOCKED]
Red / Red Advance  — gemini:gemini-3.8-flash

============================================================
CHECKPOINT 3: Assignment suite → outputs/*.json
============================================================
Suite finished.
Wrote outputs under repo outputs/

============================================================
Lab 11 complete! Check your results above.
============================================================
```

### 3.2. Kiểm thử Toàn diện Contract & Schema (`pytest`)
Thực thi kiểm tra toàn bộ test contract công khai và hợp đồng cấu trúc schema của `results.json`:
```powershell
.\.venv\Scripts\pytest.exe tests/public/test_lab_contracts.py tests/public/test_results_contract.py -v
```

**Kết quả kiểm thử:**
```text
tests/public/test_lab_contracts.py::test_detect_injection_basic PASSED                           [ 10%]
tests/public/test_lab_contracts.py::test_detect_indirect_unicode_injection_without_blocking_benign_external_data PASSED [ 20%]
tests/public/test_lab_contracts.py::test_topic_filter_blocks_off_topic PASSED                  [ 30%]
tests/public/test_lab_contracts.py::test_content_filter_redacts_secrets PASSED                  [ 40%]
tests/public/test_lab_contracts.py::test_egress_policy_blocks_sensitive_payload_and_unknown_destination PASSED [ 50%]
tests/public/test_lab_contracts.py::test_reference_boundary_requires_exact_destination_and_human_approval PASSED [ 60%]
tests/public/test_results_contract.py::test_results_json_matches_schema PASSED                  [ 70%]
tests/public/test_results_contract.py::test_safe_queries_mostly_unblocked PASSED                [ 80%]
tests/public/test_results_contract.py::test_attacks_mostly_blocked PASSED                       [ 90%]
tests/public/test_results_contract.py::test_rate_limit_blocks_excess PASSED                   [100%]

============================= 10 passed in 3.50s ==============================
```

### 3.3. Các Artifacts Đã Tạo Trong `outputs/`
| File | Dung lượng | Mô tả vai trò |
| :--- | :--- | :--- |
| `outputs/results.json` | ~5.3 KB | **BẮT BUỘC**: Kết quả phòng thủ Blue Team, chuẩn xác theo `schemas/results.schema.json` |
| `outputs/audit_log.json` | ~13.8 KB | **KHUYẾN NGHỊ**: Lưu vết 33 tương tác chi tiết (user, query, latency, layer, status) |
| `outputs/metrics.json` | ~0.4 KB | **KHUYẾN NGHỊ**: Báo cáo metrics (33 reqs, 16 blocked, 5 rate-limit hits, kèm alert) |

---

## 4. Kết Luận
- Pha 4 đã hoàn thành 100% mục tiêu, toàn bộ 10/10 test cases của hệ thống chấm công khai đạt trạng thái **PASSED**.
- Tất cả các artifacts bắt buộc cho phần phòng vệ (**Blue Team**) đã sẵn sàng trong thư mục `outputs/`.
- Sẵn sàng chuyển sang **Pha 5: Thiết kế Prompt Tấn công Red & Red Advance (Checkpoint 4)**.
