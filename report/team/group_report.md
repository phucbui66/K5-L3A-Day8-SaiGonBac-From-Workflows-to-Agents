# Báo cáo nhóm — Lab 8 LogiX-VN

Số liệu lấy từ báo cáo do công cụ lab sinh ra; phạm vi đánh giá là bộ public. Báo cáo theo cấu trúc mẫu, tối đa 5 trang ngoài phụ lục.

## 1. Thông tin nhóm

- Tên nhóm: SaiGonBac.
- Thành viên: Bùi Hữu Phúc (2A202603572), Trần Thành Nam (2A202603564); xem [TEAM.md](../../TEAM.md).
- Commit chốt : 23642e0

## 2. Mục tiêu và cách tiếp cận

Nhóm xử lý claim hoàn phí vận chuyển LogiX-VN theo cùng luật nghiệp vụ trên bốn kiến trúc, rồi lựa chọn mức tự chủ thấp nhất đáp ứng yêu cầu. Đơn hàng và policy hiện hành là nguồn sự thật; nội dung ticket là dữ liệu không tin cậy. Tiền, approval và ghi ledger thuộc quyền quyết định của code.

Bùi Hữu Phúc phụ trách Shape A (workflow), Shape B (single agent), kiểm thử nghiệp vụ/retry/crash và tổng hợp báo cáo. Trần Thành Nam phụ trách Shape C (orchestrator), handoff/trace, benchmark bốn shape, ablation, ADR và research card. Shape D (swarm) có sẵn để đối chứng. Nhóm triển khai A/B/C, kiểm tra ranh giới an toàn, rồi so sánh trên cùng corpus, seed, model, bảng giá và evaluator.

## 3. Giải pháp

### 3.1. Shape A — Workflow

[workflow.py](../../src/lab8/shapes/workflow.py) thực hiện chuỗi Python cố định: đọc claim qua `get_claim_view`; thiếu mã đơn thì `needs_info`; tra đơn theo principal, không thấy hoặc không có quyền thì `escalated`; tính quote bằng `compute_compensation`. Quote bằng 0 trả `denied`, không gọi host ghi tiền. Workflow không gọi model.

Khi quote dương, shape chuyển toàn bộ quote vào `durability.issue_refund`. Host từ chối thì shape trả `denied`; xung đột số tiền khi retry trả `failed`; thành công trả `resolved`. Kết quả giữ quote đầy đủ trong `proposal_amount`.

### 3.2. Shape B — Single agent

[single.py](../../src/lab8/shapes/single.py) dùng `run_react(max_steps=8, worker=True)` với hai tool đọc: `lookup_order` và `lookup_policy`. MockLLM có parser, không có API tool calling: callback `decide` chuyển kết quả parser thành thứ tự đọc trong tập tool giới hạn. Ticket có số ngày trễ thì đọc policy trước; trường hợp còn lại đọc đơn trước. Sau mỗi observation, callback kiểm tra scope và chọn tool còn thiếu hoặc kết thúc. Principal và mã đơn lấy từ claim đã lọc.

Sau vòng lặp, code tính quote từ order view trong database rồi gọi host. Số ngày parser đọc từ ticket chỉ ảnh hưởng thứ tự đọc, không sửa order hay quote. Hết số bước thì trả `failed`, không ghi refund. Mọi lời gọi parser, lập kế hoạch và soạn lời đáp đi qua MockLLM và được tính vào Budget; không ghi model output nguyên văn vào trace.

### 3.3. Durability, approval và trace

A/B/C truyền `crash_at` vào host và để `InjectedCrash` truyền ra ngoài; kết nối SQLite đóng bằng `finally`. Idempotency do ledger và transaction của host bảo đảm, với key `refund:{claim_id}`. Shape không ghi trực tiếp ledger và không tự trừ refund trước đó. T14 giữ proposal 150.000 VND, host trừ 30.000 VND lịch sử và ghi 120.000 VND. Quote dương vẫn đi qua host khi approval sai để có audit `policy_denied`.

Mọi nhánh trả về dùng `finish()` để đọc effect/audit thật từ ledger và ghi trace mới đúng `run_id`. A ghi trạng thái claim, kết quả scope, quote, kết quả host và quyết định; B thêm tên action/observation của ReAct. Không ghi ticket, parent context, secret, header, phone, địa chỉ, internal note hay item name. Việc bỏ field được thực hiện bằng code trước khi ghi trace.

### 3.4. Shape C, handoff/trace và đối chứng swarm

[orchestrator.py](../../src/lab8/shapes/orchestrator.py) đọc claim qua `get_claim_view`, dừng sớm nếu claim không tồn tại hoặc thiếu mã đơn. Orchestrator dùng MockLLM cho bước lập kế hoạch, rồi tạo `handoff.v1` bằng `make_handoff`: `facts={}`, scope chỉ chủ đơn hàng, chỉ cho bốn tool đọc (`lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`). Investigator chạy qua `run_readonly_investigators` với principal từ claim đã lọc; `runtime.dispatch(worker=True)` tiếp tục chặn tool ghi. Orchestrator lấy quote tính từ order/policy rồi chuyển quote đầy đủ cho host kiểm approval, trần theo đơn, idempotency và ghi ledger.

Trace C ghi trạng thái claim, handoff đã lọc, số policy, trạng thái quyền đọc, quote, kết quả host và quyết định; không ghi nguyên claim/order row, parent material hoặc model output. Test riêng xác nhận canary không xuất hiện trong trace/result; cả `make_handoff` và `dispatch` từ chối `issue_refund`/`append_audit` ở investigator.

Shape D là [swarm_reference.py](../../src/lab8/shapes/swarm_reference.py) có sẵn. D trao đổi thêm bằng MockLLM và ở seed chia hết cho 5 có thể lấy delay từ ticket thay cho order row. D trượt C-002 ở seed 55: trả `resolved` trong khi nhãn cần `denied`. Host vẫn giữ ranh giới approval/quyền ghi, nhưng không sửa quote nghiệp vụ lấy từ nguồn sai.

## 4. Kết quả

### 4.1. Benchmark bốn shape và quyết định kiến trúc

[benchmark.json](../../src/reports/benchmark.json) chứa 4 shape × 14 claim public × 5 seed `11,22,33,44,55` = 280 trial. Mỗi trial dùng SQLite riêng; cố định MockLLM, bảng giá, evaluator và mốc policy `2026-11-15`.

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean/p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| Workflow | 1,000 | 1,000 | 1,000–1,000 | 0 | 27,86 | 4,43 / 5 |
| Single | 1,000 | 1,000 | 1,000–1,000 | 0 | 38,92 | 7,07 / 8 |
| Orchestrator | 1,000 | 1,000 | 1,000–1,000 | 0 | 57,12 | 5,36 / 6 |
| Swarm | 0,986 | 0,929 | 0,929–1,000 | 0,000816 | 49,57 | 2,80 / 3 |

A/B/C đạt 70/70 trial mỗi shape; D đạt 69/70 và pass^5 = 13/14. Run id dẫn chứng: `R-workflow-C-014-s11`, `R-orchestrator-C-014-s11`, `R-swarm-C-002-s55`. Nhóm chọn **workflow** theo [decision_adr.md](decision_adr.md): cùng độ đúng public với B/C, mức tự chủ thấp nhất và cost/success thấp nhất. Chưa có kết quả hidden để xác nhận ngưỡng pass rate chính thức.

### 4.2. Kiểm thử và crash safety

Kiểm tra bằng Python 3.12.10 trên Windows:

| Kiểm tra | Kết quả |
| --- | --- |
| Cổng host, `python -m pytest -q` | `107 passed, 248 deselected` |
| Ma trận student A/B/C/D | `248 passed, 107 deselected`; mỗi shape qua 62 test có sẵn |
| Test thành viên 1 bổ sung | 46 ca trong [test_team_workflow_single.py](../../src/tests/test_team_workflow_single.py) |
| Test thành viên 2 bổ sung | 7 ca trong [test_team_orchestrator.py](../../src/tests/test_team_orchestrator.py) |
| Crash suite | [crash.json](../../src/reports/crash.json): `20/20` |
| Kiểm tra format bài nộp | `python -m lab8.check_submission`: `0 FAIL, 1 WARN`; cảnh báo do báo cáo đang sửa chưa commit |

Crash suite chung kiểm host và một ca swarm. A/B/C còn được kiểm riêng qua T11/T12 và regression ở ba vị trí `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`. Resume và replay sau crash giữ đúng một refund và một audit committed. Ca lệch nghiệp vụ cố ý của swarm được ma trận kiểm là lỗi dự kiến; benchmark vẫn tính trial đó là fail.

### 4.3. Ablation R4

Theo [research_card.md](research_card.md), nhóm giữ cố định model, prompt, 14 claim, 5 seed, giá, host và evaluator; chỉ bỏ lượt đọc `lookup_refund_history` của investigator C. [ablation_no_history.json](../../src/reports/ablation_no_history.json) cho cả hai biến thể 70/70, pass^5 = 1,000, variance = 0; cost/success giảm 57,12 → 47,84 VND, khoảng 16,3% tính từ số liệu gốc.

Host vẫn tự kiểm tra tổng refund trong transaction, kể cả T13/T14. Kết quả ủng hộ hypothesis rằng lượt đọc history không cần cho quyết định hiện tại trên public corpus; không chứng minh toàn bộ chênh lệch A/B/C/D do topology. Lệnh ablation và run id được lưu trong research card.

## 5. Khó khăn và cách xử lý

Môi trường ban đầu có Python 3.12.10 nhưng chưa có pytest (`python -m pip show pytest` báo package không tồn tại). Nhóm tạo môi trường bằng `python -m venv .venv`. Lần cài dependencies trong sandbox bị treo; sau khi dừng tiến trình và cài lại ngoài sandbox qua cơ chế phê duyệt, pytest 8.4.2 và PyYAML 6.0.3 được cài từ requirements của đề.

MockLLM không có API tool calling, nên B dùng callback giới hạn tool/đối số và chỉ dùng parser để chọn thứ tự đọc. Với C, handoff được lọc bằng allowlist và investigator không nhận tool ghi; tóm tắt bằng model không thay cho lọc dữ liệu. Nhóm dùng test ticket injection, canary và crash để kiểm tra các ranh giới này.

Các lệnh kiểm tra/tái hiện kết quả, chạy trong [src/](../../src/) sau khi activate môi trường:

```text
python -m pytest -q
python -m pytest -q -m student -o addopts= --tb=short
python -m lab8.crash_test --count 20 --out reports/crash.json
python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
python -m lab8.check_submission
```

## 6. Tổng hợp đóng góp

| Thành viên | Phần việc | Bằng chứng |
| --- | --- | --- |
| Bùi Hữu Phúc — 2A202603572 | A/B, nghiệp vụ, approval, retry/crash, trace, 46 ca test và tổng hợp báo cáo | Commit kỹ thuật `678d25c`; [báo cáo cá nhân](2A202603572_BuiHuuPhuc.md) |
| Trần Thành Nam — 2A202603564 | C, handoff/trace, 7 ca test, benchmark bốn shape, ablation, ADR và research card | Commit kỹ thuật `a4ebcf4`; [báo cáo cá nhân](2A202603564_TranThanhNam.md) |

Thông tin chi tiết nằm trong [TEAM.md](../../TEAM.md) và báo cáo cá nhân. Nhóm cần đồng bộ commit chốt trong ADR và bảng đóng góp trước khi nộp.

## 7. Hạn chế và hướng cải thiện

Public corpus chỉ có 14 claim; chưa suy ra hidden pass rate. MockLLM, VND mô phỏng và latency `unavailable` không đại diện model hay chi phí sản xuất. Các shape cùng dữ liệu/seed/giá nhưng khác prompt, số model call và đường tool, nên so sánh giữa shape có yếu tố gây nhiễu; steps là số event trace và phụ thuộc cách ghi log.

Ablation chỉ kiểm tra một lượt đọc tool. Cần thêm claim có các nhánh điều tra độc lập và ablation chỉ thay một thành phần, đo pass^5 và cost/success trên corpus mới. Quyết định có thể đổi nếu kiến trúc nhiều agent cải thiện độ đúng/ổn định đủ để bù chi phí khi đo cùng điều kiện. Nếu logging ở ngoài `Trace`, phải quét mọi đích ghi và stdout; scanner hiện chỉ thấy trace và các file cùng tiền tố run id trong cùng thư mục.

## 8. Tài liệu tham khảo

- [Đề bài](../../README.md), [luật nghiệp vụ](../../docs/business_rules.md), [output/trace contract](../../docs/contracts.md), [kiến trúc](../../docs/architecture.md), [metric](../../docs/metrics.md) và [quy định nộp bài](../../SUBMISSION.md).
- Code host/runtime/MockLLM và swarm tham chiếu của repo đề bài; các shape sử dụng adapter/helper được cung cấp.
- [Zhang et al., *Rethinking the Evaluation of Efficiency Methods for Multi-Agent Systems*, EMNLP 2026](https://aclanthology.org/2026.emnlp-main.798/): nguồn nghiên cứu R4 trong research card; nhóm vận dụng nguyên tắc kiểm soát biến và giới hạn kết luận trong phạm vi phép đo LogiX-VN.
