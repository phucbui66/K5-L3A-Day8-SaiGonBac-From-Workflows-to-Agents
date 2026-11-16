# Team

## Thông tin nhóm

- Tên nhóm:
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-TenNhom-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt:

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 |     Bùi Hữu Phúc   |  2A202603572    | phucbh.66.24@gmail.com      | phucbui66       | Workflow + Single; nghiệp vụ, retry/crash; tích hợp và báo cáo nhóm |
|   2 |   Trần Thành Nam     |      |   thanhnam120304@gmail.com    |        | Orchestrator; handoff/trace; benchmark, ablation, ADR và research card |

Gợi ý chia vai:

- Nhóm 2 người: phân công như bảng thành viên ở trên; họ tên, MSSV và tài khoản cần điền theo thông tin thực tế.
- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Bùi Hữu Phúc | Đã triển khai workflow, single và test nghiệp vụ/retry/crash; soạn phần báo cáo được giao |  | [test_team_workflow_single.py](src/tests/test_team_workflow_single.py): 11 hàm test, 46 ca; gồm crash cả 3 vị trí, approval sai claim, top-up/retry và trace |

Đây là bảng đang hoàn thiện. Thành viên 2 bổ sung một dòng với commit và test thực tế của mình; điền định danh thật của thành viên 1 trước khi nộp.

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Bản nháp thành viên 1: [member1_draft.md](report/team/member1_draft.md). Cần điền định danh, tự đánh giá và đổi tên đúng MSSV/họ tên trước khi nộp; giữ nguyên file mẫu.

Sao chép mục dưới đây cho từng thành viên.

## MSSV_HoTen

### Phần việc phụ trách

-

### Đóng góp cụ thể

-

### Bằng chứng

-
