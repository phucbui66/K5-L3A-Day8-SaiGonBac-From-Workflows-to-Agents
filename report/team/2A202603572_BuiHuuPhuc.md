# Báo cáo cá nhân — Thành viên 1 (bản nháp)

## 1. Thông tin thành viên

- Họ tên: Bùi Hữu Phúc  
- MSSV: 2A202603572
- Email: phucbh.66.24@gmail.com  
- GitHub: phucbui66

## 2. Phần việc phụ trách

- CP2: Shape A [workflow.py](../../src/lab8/shapes/workflow.py) và Shape B [single.py](../../src/lab8/shapes/single.py).
- CP3–CP4: approval, trần hoàn theo đơn, retry/crash và trace của A/B; viết test bổ sung, chạy smoke/cổng host/ma trận student/crash suite.
- CP5: bàn giao A/B, bằng chứng crash, thông tin đóng góp và phần báo cáo nhóm được giao. Benchmark hai shape dùng để tự kiểm; benchmark bốn shape, ADR, research card và ablation do thành viên 2 chủ trì.

## 3. Đóng góp cụ thể

- Commit chính (hash): chưa commit; điền hash của commit chứa thay đổi này sau khi review và commit thật.
- Workflow dùng chuỗi bước cố định và không gọi model. Single dùng vòng ReAct tối đa 8 bước, worker chỉ có tool đọc; parser MockLLM chỉ ảnh hưởng thứ tự đọc policy/đơn.
- Cả hai tính tiền từ order và policy bằng code, chuyển quote đầy đủ vào host để host trừ refund lịch sử/cap approval; giữ `proposal_amount` bằng quote ban đầu.
- Khi quote dương nhưng approval không hợp lệ, vẫn gọi host để ghi `policy_denied`. Không tự tạo approval hay ghi trực tiếp ledger.
- Truyền `crash_at`, không bắt `InjectedCrash`; đóng kết nối bằng `finally`. Dùng `finish()` để lấy effect/audit từ ledger và ghi trace mới ở mọi nhánh trả về.
- Thêm [test_team_workflow_single.py](../../src/tests/test_team_workflow_single.py): 11 hàm test, 46 ca sau tham số hóa. Nổi bật: `test_every_supported_crash_window_resumes_exactly_once`, `test_top_up_retry_keeps_full_proposal_and_one_net_effect`, `test_approval_denial_is_recorded_by_the_host`, `test_single_ignores_model_money_and_counts_actual_model_calls`.

## 4. Bằng chứng

Các lệnh chạy trong [src/](../../src/) bằng Python của môi trường `.venv`:

| Lệnh | Kết quả thực tế |
| --- | --- |
| `python -m lab8.smoke` | `lab8.smoke pass` |
| `python -m pytest -q tests/test_team_workflow_single.py` | `46 passed` |
| `python -m pytest -q` | `100 passed, 248 deselected` |
| `python -m pytest -q -m student -o addopts= --tb=no --no-summary` | `186 passed, 62 failed, 100 deselected`; lỗi chỉ ở orchestrator stub. A và B mỗi shape qua 62 ca public. |
| `python -m lab8.crash_test --count 20 --out reports/crash.json` | `crash_safety 20/20`, [bằng chứng JSON](../../src/reports/crash.json) |
| `python -m lab8.benchmark --shapes workflow,single --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/member1_benchmark.json` | Mỗi shape `70/70` trial, pass rate/pass^5 = `1.0`, [bằng chứng JSON](../../src/reports/member1_benchmark.json) |

Run id ví dụ: `R-workflow-C-001-s11`, `R-single-C-001-s11`. Đây là số liệu public, chưa có điểm hidden. Benchmark hai shape chưa thay thế benchmark bốn shape bắt buộc của nhóm.

`python -m lab8.check_submission` hiện ra `10 FAIL, 1 WARN`: A/B và crash được xác nhận; còn C, benchmark bốn shape, ADR/research card và commit cuối để hoàn tất bài nhóm.

## 5. Công cụ AI đã dùng

- Công cụ: Codex, hỗ trợ đọc yêu cầu, đề xuất và triển khai code/test, chạy kiểm thử, tổng hợp bằng chứng và soạn bản nháp báo cáo.
- Kiểm tra đã thực hiện bằng công cụ: chạy các lệnh ở mục 4, đối chiếu trace/ledger/audit trong test, dùng output JSON của công cụ để báo số liệu; không sửa tay JSON kết quả hoặc thước đo của đề.
- Thành viên cần tự đọc code, chạy lại các lệnh và giải thích được ranh giới code/model, quote đầy đủ, allowlist trace và ledger sau crash. Chưa ghi nhận việc thành viên đã tự kiểm tra trực tiếp.

## 6. Tự đánh giá

Chờ thành viên tự điền sau khi đọc code và tự chạy lại kiểm thử. Nêu rõ phần hiểu được, phần còn hạn chế và việc phối hợp với thành viên 2; không dùng kết quả kiểm thử do trợ lý chạy để thay cho tự đánh giá cá nhân.
