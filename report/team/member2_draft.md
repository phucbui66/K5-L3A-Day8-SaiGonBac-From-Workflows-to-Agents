# Phần thành viên 2 để ghép vào báo cáo nhóm

## 3.4. Shape C, handoff/trace và đối chứng swarm

[orchestrator.py](../../src/lab8/shapes/orchestrator.py) đọc claim qua `get_claim_view`, dừng sớm nếu claim không tồn tại hoặc thiếu mã đơn. Orchestrator dùng MockLLM để ghi nhận bước lập kế hoạch, rồi tạo `handoff.v1` bằng `make_handoff`: `facts={}`, scope chỉ chủ đơn hàng, chỉ cho bốn tool đọc (`lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`). Investigator chạy qua `run_readonly_investigators` với principal từ claim đã lọc. `runtime.dispatch(worker=True)` tiếp tục chặn tool ghi. Orchestrator lấy quote do tool tính từ order view và policy hiện hành, rồi gọi `durability.issue_refund` với quote đầy đủ. Host kiểm tra approval, trần theo đơn, idempotency và là bên ghi ledger duy nhất. C chuyển tiếp `crash_at` và để `InjectedCrash` lan ra; chạy lại không tạo refund thứ hai.

Trace C chỉ ghi trạng thái claim, handoff đã lọc, số policy, trạng thái quyền đọc, quote, kết quả host và quyết định. Không ghi ticket, `parent_secret`, raw header, phone, address, internal note, item name, nguyên claim/order row hoặc model output. Mọi nhánh trả về dùng `finish()` để tạo trace mới đúng run id. Test riêng xác nhận canary không xuất hiện trong trace/result và cả `make_handoff` lẫn `dispatch` từ chối `issue_refund`/`append_audit` ở investigator.

Shape D là [swarm_reference.py](../../src/lab8/shapes/swarm_reference.py) có sẵn. Nó trao đổi thêm bằng MockLLM và ở seed chia hết cho 5 có thể lấy delay từ ticket thay cho order row. Trong benchmark, D trượt C-002 ở seed 55: báo `resolved` trong khi nhãn cần `denied`. Host vẫn giữ ranh giới approval và quyền ghi, nhưng không sửa được quote nghiệp vụ bị model lấy từ nguồn sai.

## 4. Kết quả benchmark bốn shape

Chạy `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json` trong `src/`. Mỗi trial dùng SQLite riêng, cùng 14 claim, 5 seed, MockLLM, bảng giá và evaluator. Kết quả ở [benchmark.json](../../src/reports/benchmark.json):

| Shape | Pass rate | Pass^5 | Seed min–max | Variance | Cost/success (VND mô phỏng) | Mean steps |
| --- | ---: | ---: | --- | ---: | ---: | ---: |
| Workflow | 1,000 | 1,000 | 1,000–1,000 | 0 | 27,86 | 4,43 |
| Single | 1,000 | 1,000 | 1,000–1,000 | 0 | 38,92 | 7,07 |
| Orchestrator | 1,000 | 1,000 | 1,000–1,000 | 0 | 57,12 | 5,36 |
| Swarm | 0,986 | 0,929 | 0,929–1,000 | 0,000816 | 49,57 | 2,80 |

Ma trận student: `248 passed`; toàn bộ test mặc định: `106 passed, 248 deselected`; test riêng C: `6 passed`. Crash C được thử ở `before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`, cả ba đều retry ra đúng một refund. Crash suite chung ở [crash.json](../../src/reports/crash.json) là 20/20. `check_submission` đọc đúng C và benchmark bốn shape.

Ablation R4 giữ cố định model, prompt, 14 claim, 5 seed, giá, host và evaluator; chỉ bỏ lượt đọc `lookup_refund_history` của investigator C. [ablation_no_history.json](../../src/reports/ablation_no_history.json) cho cả hai biến thể 70/70, pass^5 1,000, variance 0; cost/success giảm 57,12 → 47,84 VND (−16,2%). Host vẫn tự kiểm tra tổng hoàn tiền trong transaction, kể cả T13/T14. Điều này chứng minh riêng lượt đọc history không cần cho quyết định hiện tại trên public corpus; không chứng minh toàn bộ chênh lệch A/B/C/D do topology.

## 7. Hạn chế và hướng cải thiện

Public corpus chỉ có 14 claim; chưa suy ra được hidden pass rate. MockLLM, VND mô phỏng và latency `unavailable` không đại diện model hay chi phí sản xuất. Các shape cùng dữ liệu/seed/giá nhưng khác prompt, số model call và đường tool nên so sánh giữa shape có yếu tố gây nhiễu. Có thể thử thêm claim điều tra độc lập và ablation chỉ thay một thành phần, đo pass^5 và cost/success trên corpus mới. Nếu triển khai logging khác `Trace`, phải quét mọi đích ghi và stdout; scanner chỉ thấy file trace và file cùng tiền tố run id trong cùng thư mục.

## 8. Tài liệu tham khảo

- [Business rules](../../docs/business_rules.md), [contracts](../../docs/contracts.md), [architecture](../../docs/architecture.md), [metrics](../../docs/metrics.md) và [submission](../../SUBMISSION.md) của lab.
- [Zhang et al., *Rethinking the Evaluation of Efficiency Methods for Multi-Agent Systems*, EMNLP 2026](https://aclanthology.org/2026.emnlp-main.798/). Paper chỉ ra kết luận tiết kiệm MAS phụ thuộc prompt/topology/runtime thí nghiệm; nhóm dùng nguyên tắc kiểm soát biến, không áp kết quả paper thành claim về LogiX-VN.
