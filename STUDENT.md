# Thông tin học viên

- Họ và tên: Ngô Trí Anh
- MSHV: 2A202604053
- Lớp (L2 / L3 / khác): L2B
- Tài khoản GitHub: AnhNgo1702
- Repo bài nộp (URL): https://github.com/AnhNgo1702/K5-L23-Day08-NgoTriAnh-2A202604053-AI-Assisted-Labeling
- Commit chốt bài (hash):
- Máy: Windows 11, không có GPU (chạy CPU)
- CVAT (Docker): v2.74.1

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 386 / 18 / 26 (mẫu số: 435 box máy); added: 47 (mẫu số: 451 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 74.6% / 84.6% (mẫu số: 52 đối tượng gold)
- Bắt bẫy: 11/12 = 91.7%
- Thời gian: gán tay 5.0 phút/ảnh; duyệt có pre-label 3.75 phút/ảnh
- Lỗi QC: 0.0% theo ảnh (0/8), 0.0% theo đối tượng (0/147)
- **Quyết định:** Release — lý do một dòng: Không còn khoảng trống guideline, tỉ lệ ảnh lỗi QC 0.0% <= 2% và tỉ lệ bắt bẫy đạt 91.7% >= 80%.
