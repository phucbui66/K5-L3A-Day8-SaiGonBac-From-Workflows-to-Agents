# Decision record

- Shape chọn: workflow
- Commit hash: 678d25c (base; cập nhật theo commit chốt trước khi nộp)
- Run id dẫn chứng: R-workflow-C-014-s11; R-orchestrator-C-014-s11; R-swarm-C-002-s55
- Pass rate / pass^5 / seed min-max / variance: workflow 1,000 / 1,000 / 1,000–1,000 / 0; xem src/reports/benchmark.json
- Cost per success (mô phỏng, VND): workflow 27,86; single 38,92; orchestrator 57,12; swarm 49,57
- Crash safety: 20/20 theo src/reports/crash.json; C qua 3/3 cửa sổ crash và retry trong test_team_orchestrator.py
- Secret leak / approval bypass: 0 trong 70 trial C công khai và test handoff/trace; các ca approval sai đi qua host, không có refund trái phép

## Vì sao shape này

Trên 14 claim public × 5 seed, workflow, single và orchestrator đều đạt 70/70, pass^5 1,000, seed variance 0. Workflow có cost/success thấp nhất (27,86 VND mô phỏng). Swarm đạt 69/70, pass^5 13/14, seed variance 0,000816; ở C-002 seed 55 nó dùng delay trong ticket thay vì order row. Các claim hoàn tiền hiện chủ yếu theo trình tự đọc dữ liệu, tính quote, gọi host; chưa có bằng chứng rằng thêm agent tăng chất lượng. Vì vậy chọn workflow làm shape nộp hidden, giữ C làm đối chứng kiến trúc.

## Có đáng đổi lấy multi-agent không

Với corpus này, không: C cùng độ đúng public nhưng chi phí hơn gấp đôi A. Quyết định có thể đổi nếu một corpus mới có tác vụ điều tra độc lập đủ phức tạp, C cải thiện pass^5 hoặc giảm lỗi có ý nghĩa khi giữ nguyên model, tool, seed, evaluator và ngân sách; cần đo lại cost/success trên cùng tập. Ablation bỏ `lookup_refund_history` của C giảm 9,28 VND mỗi success mà không đổi kết quả public, nhưng vẫn chưa làm C rẻ hơn A.

## Trace và bằng chứng

Shape C ghi sự kiện claim có/không có order id, handoff đã lọc, tóm tắt investigator, quote, host outcome và decision. Handoff chỉ có contract fields, `facts={}` và bốn tool đọc; investigator bị `make_handoff` lẫn `runtime.dispatch(worker=True)` chặn tool ghi. Trace không ghi ticket, parent context, secret, header, phone, address, internal note, item name hay nguyên order view. `finish()` tạo trace mới ở mọi nhánh trả về; scanner kiểm tra `trace_path` và các file cùng tiền tố run id. Nếu log ở ngoài phạm vi scanner, không thể lấy kết quả scanner làm bằng chứng sạch; phải gom hoặc kiểm tra thêm mọi đích ghi log và stdout trước khi kết luận.

## Giới hạn

MockLLM không phải model sản xuất; cost là VND mô phỏng, latency `unavailable`. Public 14 claim không bảo đảm hidden pass rate. So sánh A/B/C/D cùng corpus, seed và giá nhưng khác prompt, số model call và cấu trúc tool; do đó không quy toàn bộ chênh lệch cho topology. `Commit hash` phía trên là commit nền, cần thay bằng hash commit chốt của nhóm.
