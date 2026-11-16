# Báo cáo cá nhân — 2A202603564_TranThanhNam

## 1. Thông tin thành viên

- Họ tên: Trần Thành Nam
- MSSV: 2A202603564

## 2. Phần việc phụ trách

 Shape C (orchestrator và investigator), ranh giới handoff/trace, kiểm thử retry và crash cho C. Benchmark công khai trên bốn shape A/B/C/D với năm seed, thực hiện một ablation, điền `research_card.md` và `decision_adr.md`, đồng thời soạn phần Shape C và đối chứng swarm ở mục 3, bảng benchmark ở mục 4, mục 7–8 cho báo cáo nhóm.

## 3. Đóng góp cụ thể

- Triển khai [orchestrator.py](../../src/lab8/shapes/orchestrator.py): orchestrator đọc claim, tạo handoff chỉ chứa field cho phép, giao các phép đọc cho investigator, lấy quote tính từ order/policy và gọi host `issue_refund` với quote đầy đủ. Host quyết định approval, trần tiền theo đơn, idempotency và ghi ledger.
- Siết [handoff.py](../../src/lab8/handoff.py): `sanitize` lọc cả field lạ trong `facts` và từ chối mọi tool ngoài tập read-only. Investigator không nhận `issue_refund` hoặc `append_audit`.
- Thêm [test_team_orchestrator.py](../../src/tests/test_team_orchestrator.py): kiểm tra handoff/trace không chứa canary hoặc parent material; từ chối tool ghi; retry sau crash tại `before_effect`, `after_effect_before_checkpoint` và `after_checkpoint` chỉ ghi một refund.
- Tạo biến thể [orchestrator_no_history.py](../../src/lab8/shapes/orchestrator_no_history.py) cho ablation: chỉ bỏ lượt đọc `lookup_refund_history`; giữ nguyên đường tính quote và host.
- Chạy benchmark bốn shape và ablation; ghi kết quả vào [research_card.md](research_card.md), [decision_adr.md](decision_adr.md) và [phần báo cáo nhóm để ghép](member2_draft.md). ADR chọn `workflow` vì trên public corpus A/B/C cùng đạt 70/70 nhưng A có cost/success thấp nhất.
- Commit phần việc: chưa có commit riêng trong working tree hiện tại; cần commit trước khi nộp và ghi hash thực tế vào đây, `TEAM.md` và ADR.

## 4. Bằng chứng

| Bằng chứng | Kết quả |
| --- | --- |
| `python -m pytest -q tests/test_team_orchestrator.py` | 7 passed |
| `python -m pytest -q` | 107 passed, 248 deselected |
| `python -m pytest -q -m student -o addopts= --tb=short` | 248 passed, 107 deselected |
| `python -m lab8.check_submission` | 0 FAIL, 1 WARN do thay đổi chưa commit |
| [benchmark.json](../../src/reports/benchmark.json), 14 claim × 5 seed | Workflow, single, orchestrator: 70/70; swarm: 69/70, sai C-002 seed 55 |
| [ablation_no_history.json](../../src/reports/ablation_no_history.json) | Hai biến thể C đều 70/70; cost/success 57,12 → 47,84 VND mô phỏng |
| [crash.json](../../src/reports/crash.json) | Crash suite chung 20/20; test riêng C qua ba cửa sổ crash |

Run id có thể tra trong benchmark: `R-orchestrator-C-014-s11` cho ca hoàn phần chênh lệch, `R-swarm-C-002-s55` cho lỗi đối chứng. Số tiền là cost mô phỏng theo cấu hình lab; latency là `unavailable`.

## 5. Công cụ AI đã dùng

Sử dụng ChatGPT/Codex để hỗ trợ đọc contract, viết bản triển khai C, gợi ý test, tổng hợp kết quả benchmark và soạn bản nháp báo cáo. Tự kiểm tra đầu ra bằng các lệnh test và benchmark nêu trên, đọc JSON kết quả, đối chiếu hành vi với `docs/contracts.md` và chỉ dùng số liệu do công cụ lab sinh ra. Nguồn nghiên cứu cho research card là [Zhang et al., EMNLP 2026](https://aclanthology.org/2026.emnlp-main.798/); kết luận paper được giới hạn đúng phạm vi và được kiểm tra bằng ablation riêng của nhóm.

## 6. Tự đánh giá

Phần được giao về C, handoff/trace, test, benchmark, ablation và bản nháp báo cáo đã hoàn thành trên public corpus. Điểm mạnh của cách làm là tách quyền đọc của investigator khỏi quyền ghi tiền của host và dùng cùng claim/seed/giá để so sánh. Giới hạn là chưa có kết quả hidden; MockLLM không đại diện model sản xuất; ablation chỉ kiểm tra một lượt đọc tool và không chứng minh toàn bộ chênh lệch giữa A/B/C/D do topology. Trước khi nộp cần ghép nội dung vào báo cáo nhóm, commit phần việc, cập nhật hash và tự rà soát lại thông tin cá nhân.
