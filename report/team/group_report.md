# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Bản hiện tại ghi phần việc thành viên 1 đã triển khai và kiểm thử; phần thành viên 2 được đánh dấu chờ bổ sung. Chưa phải bản nộp cuối.

## 1. Thông tin nhóm

- Tên nhóm: SaiGonBac
- Thành viên: Bùi Hữu Phúc , Trần Thành Nam
- Commit chốt (trùng [decision_adr.md](decision_adr.md)): chưa chốt, thay đổi hiện chưa commit.

## 2. Mục tiêu và cách tiếp cận

Nhóm xử lý claim hoàn phí vận chuyển LogiX-VN theo cùng luật nghiệp vụ trên bốn kiến trúc, rồi lựa chọn mức tự chủ thấp nhất đáp ứng yêu cầu. Đơn hàng và policy hiện hành là nguồn sự thật; nội dung ticket là dữ liệu không tin cậy. Tiền, approval và ghi ledger thuộc quyền quyết định của code.

Thành viên 1 triển khai Shape A (workflow), Shape B (single agent), kiểm thử nghiệp vụ/retry/crash và tổng hợp báo cáo. Thành viên 2 phụ trách Shape C (orchestrator), handoff, benchmark bốn shape, ablation, ADR và research card. Shape D (swarm) có sẵn để đối chứng. Hai người cùng đọc contract, kiểm tra kết quả và tự viết báo cáo cá nhân.

## 3. Giải pháp

### 3.1. Shape A — Workflow

[workflow.py](../../src/lab8/shapes/workflow.py) thực hiện chuỗi Python cố định: đọc claim qua `get_claim_view`; thiếu mã đơn thì `needs_info`; tra đơn theo principal, không thấy hoặc không có quyền thì `escalated`; tính quote bằng `compute_compensation`. Quote bằng 0 trả `denied`, không gọi host ghi tiền. Workflow không gọi model.

Khi quote dương, shape chuyển toàn bộ quote vào `durability.issue_refund`. Host từ chối thì shape trả `denied`; xung đột số tiền khi retry trả `failed`; thành công trả `resolved`. Kết quả luôn giữ quote đầy đủ trong `proposal_amount`.

### 3.2. Shape B — Single agent

[single.py](../../src/lab8/shapes/single.py) dùng `run_react(max_steps=8, worker=True)` với hai tool đọc: `lookup_order` và `lookup_policy`. MockLLM có parser, không có API tool calling: callback `decide` chuyển kết quả parser thành thứ tự đọc trong tập tool giới hạn. Ticket có số ngày trễ thì đọc policy trước; trường hợp còn lại đọc đơn trước. Sau mỗi observation, callback kiểm tra scope và chọn tool còn thiếu hoặc kết thúc. Principal và mã đơn luôn lấy từ claim đã lọc.

Sau vòng lặp, code tính quote từ order view trong database rồi gọi host như workflow. Số ngày parser đọc từ ticket chỉ ảnh hưởng thứ tự đọc, không sửa order hay quote. Nếu hết số bước thì trả `failed`, không ghi refund. Mọi lần gọi parser, lập kế hoạch và soạn lời đáp đều qua MockLLM và được tính vào Budget. Model output không được ghi nguyên văn vào trace.

### 3.3. Durability, approval và trace của A/B

Cả hai shape truyền `crash_at` vào host và để `InjectedCrash` truyền ra ngoài. Kết nối SQLite luôn đóng bằng `finally`. Idempotency do ledger và transaction của host bảo đảm; key là `refund:{claim_id}`. Không ghi trực tiếp ledger và không tự trừ phần claim trước đã nhận. Ví dụ T14 giữ proposal 150.000 VND, host trừ 30.000 VND lịch sử và chỉ ghi 120.000 VND. Quote dương vẫn phải đi qua host khi approval sai để có audit `policy_denied`.

Các nhánh trả về dùng `finish()` để đọc effect/audit thật từ ledger và ghi trace mới đúng `run_id`. Trace của A ghi trạng thái đọc claim, kết quả scope, quote, kết quả host và quyết định; B thêm tên action/observation của vòng ReAct. Không ghi ticket, parent context, secret, header, phone, địa chỉ, internal note hay item name. Việc bỏ field được thực hiện bằng code trước khi ghi trace.

### 3.4. Shape C, handoff và đối chứng D

Chờ thành viên 2 triển khai orchestrator, mô tả handoff allowlist và bổ sung so sánh với swarm. Hiện orchestrator còn stub; chưa xác nhận hoàn tất CP2–CP4 cho C.

## 4. Kết quả

Kết quả dưới đây lấy từ [member1_benchmark.json](../../src/reports/member1_benchmark.json), do công cụ benchmark tạo trên 14 claim public × 5 seed `11,22,33,44,55` × 2 shape. Đây là bằng chứng tạm cho A/B; benchmark bốn shape bắt buộc khi nộp sẽ do thành viên 2 tạo theo [SUBMISSION.md](../../SUBMISSION.md).

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean/p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| Workflow | 1,000 | 1,000 | 1,000–1,000 | 0 | 27,86 | 4,43 / 5 |
| Single | 1,000 | 1,000 | 1,000–1,000 | 0 | 38,92 | 7,07 / 8 |
| Orchestrator | Chờ thành viên 2 | — | — | — | — | — |
| Swarm | Chờ benchmark bốn shape | — | — | — | — | — |

Run id mẫu: `R-workflow-C-001-s11`, `R-single-C-001-s11`; các trial còn lại nằm trong JSON. Kết quả này không xác nhận ngưỡng hidden; chưa có dữ liệu hidden và chưa chốt shape trong ADR.

Kiểm thử thực tế bằng Python 3.12.10 trên Windows:

| Kiểm tra | Kết quả |
| --- | --- |
| Smoke | `lab8.smoke pass` |
| Cổng host, `python -m pytest -q` | `100 passed, 248 deselected` |
| Test thành viên 1 bổ sung | `46 passed` |
| Ma trận student A/B/C/D | `186 passed, 62 failed, 100 deselected`; 62 lỗi đều thuộc orchestrator stub. A/B mỗi shape qua 62 test có sẵn. |
| Crash suite | [crash.json](../../src/reports/crash.json): `20/20` |

Crash suite kiểm host và một ca swarm; độ an toàn của A/B còn được kiểm riêng bởi T11/T12 và test mới ở cả ba vị trí `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`. Resume và replay sau crash giữ đúng một refund và một audit committed.

Chờ thành viên 2 hoàn thiện [decision_adr.md](decision_adr.md), [research_card.md](research_card.md), phép đo bốn shape và một ablation trước khi chốt kết luận. Kiểm tra `python -m lab8.check_submission` hiện ra `10 FAIL, 1 WARN`: workflow/single và crash đã đạt; các lỗi còn lại là orchestrator stub, thiếu benchmark bốn shape, ADR/research card chưa điền; cảnh báo do thay đổi chưa commit.

## 5. Khó khăn và cách xử lý

Môi trường ban đầu có Python 3.12.10 nhưng chưa có pytest (`python -m pip show pytest` báo package không tồn tại). Đã tạo môi trường bằng `python -m venv .venv`. Lần cài dependencies trong sandbox bị treo, đã dừng tiến trình đó và chạy lại lệnh cài ngoài sandbox qua cơ chế phê duyệt; cài thành công pytest 8.4.2 và PyYAML 6.0.3 từ requirements của đề.

Lệnh tái hiện phần kiểm tra thành viên 1, chạy trong [src/](../../src/) sau khi activate môi trường:

```text
python -m pytest -q tests/test_team_workflow_single.py
python -m pytest -q
python -m pytest -q -m student -o addopts= --tb=no --no-summary
python -m lab8.crash_test --count 20 --out reports/crash.json
python -m lab8.benchmark --shapes workflow,single --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/member1_benchmark.json
```

Ma trận toàn nhóm còn đỏ ở C vì chưa triển khai; cần tích hợp phần thành viên 2, chạy lại và cập nhật kết quả cuối. Chờ bổ sung khó khăn của thành viên 2.

## 6. Tổng hợp đóng góp

Thành viên 1 đã có code A/B và [test_team_workflow_single.py](../../src/tests/test_team_workflow_single.py) với 46 ca sau tham số hóa, bao gồm ticket injection, scope, approval, trần theo đơn, conflict khi retry, trace và ba cửa sổ crash. [TEAM.md](../../TEAM.md) ghi phạm vi đóng góp; hiện chưa có commit cho thay đổi này và chưa điền danh tính thật.

Bản nháp cá nhân của thành viên 1: [member1_draft.md](member1_draft.md). Cần điền định danh và đổi tên đúng `MSSV_HoTen.md` trước khi nộp; giữ nguyên [file mẫu](MSSV_HoTen.md). Thành viên 2 tự bổ sung code, test, commit và báo cáo của mình.

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable`, cost là VND mô phỏng. A/B mới được xác nhận trên public và các regression bổ sung; không suy ra hidden pass rate hay chất lượng trên model thật. Hiện chưa đủ bốn shape để quyết định kiến trúc cuối. Thành viên 2 bổ sung giới hạn của phép đo/ablation và điều kiện đổi quyết định.

## 8. Tài liệu tham khảo

- [Đề bài](../../README.md), [luật nghiệp vụ](../../docs/business_rules.md), [output/trace contract](../../docs/contracts.md), [kiến trúc](../../docs/architecture.md), [metric](../../docs/metrics.md) và [quy định nộp bài](../../SUBMISSION.md).
- Code host/runtime/MockLLM có sẵn của repo đề bài; A/B sử dụng trực tiếp các adapter và helper được cung cấp.
- Thành viên 2 bổ sung nguồn nghiên cứu đã đọc cho research card; chưa khai nguồn ngoài chưa sử dụng.
