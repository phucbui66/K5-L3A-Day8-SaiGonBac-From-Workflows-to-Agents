# Research card

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R4 |
| SOURCE | Nguồn và loại bằng chứng | Zhang et al., *Rethinking the Evaluation of Efficiency Methods for Multi-Agent Systems*, EMNLP 2026, [ACL Anthology](https://aclanthology.org/2026.emnlp-main.798/); nghiên cứu benchmark có đối chứng. |
| CLAIM | Kết luận trong phạm vi nguồn | Hiệu quả tiết kiệm của phương pháp multi-agent có thể bị đánh giá quá cao khi prompt hoặc topology ban đầu khác nhau; kết quả có thể phụ thuộc setup. |
| BOUNDARY | Giới hạn | Nguồn thử các phương pháp tối ưu MAS trên benchmark và runtime của paper; không trực tiếp đo nghiệp vụ hoàn tiền LogiX-VN. |
| HYPOTHESIS | Dự đoán trước ablation | Với Shape C, bỏ một lượt đọc `lookup_refund_history` sẽ giữ pass rate và pass^5 trên public corpus, đồng thời giảm cost/success khoảng 10 VND. Host vẫn tự kiểm tra trần tiền trong transaction. |
| METHOD | Biến cố định và biến thay đổi | Cố định MockLLM, prompt, order/policy/host, 14 claim, 5 seed `11,22,33,44,55`, giá, evaluator và SQLite riêng mỗi trial. Chỉ đổi danh sách tool investigator: bật/tắt `lookup_refund_history`; đối chiếu `orchestrator` với `orchestrator_no_history`. |
| FAILURE / METRIC | Lỗi dự kiến và cách đo | Nếu bỏ history gây sai ở T13/T14 hoặc retry thì proposal/refund mismatch. Đo pass rate, pass^5, seed variance và cost/success từ cùng `lab8.benchmark`. |

## Kết quả ablation

- Lệnh đã chạy (trong `src/`): `python -m lab8.benchmark --shapes orchestrator,orchestrator_no_history --extra-shape orchestrator_no_history=lab8.shapes.orchestrator_no_history:run_claim --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/ablation_no_history.json`
- Run id dẫn chứng: `R-orchestrator-C-014-s11`; file [ablation_no_history.json](../../src/reports/ablation_no_history.json).
- Kết quả: hypothesis đúng trên public corpus. Cả hai đạt 70/70, pass^5 = 1,000, variance = 0; cost/success giảm từ 57,12 xuống 47,84 VND mô phỏng, tức 9,28 VND (16,2%). T13/T14 không sai vì host tự đọc tổng hoàn tiền khi ghi.
- Giới hạn: một ablation về lượt đọc tool, chưa chứng minh nhân quả của riêng topology A/B/C/D vì các shape còn khác prompt và số model call. MockLLM không phải model sản xuất; latency là `unavailable`. Hidden cases chưa được đo.
