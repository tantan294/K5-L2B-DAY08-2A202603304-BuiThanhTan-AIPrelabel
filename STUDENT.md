# Thông tin học viên

- Họ và tên: Bùi Thanh Tân
- MSHV: 202603304

- Lớp (L2 / L3 / khác): L2B
- Tài khoản GitHub: tantan294
- Repo bài nộp (URL): [tantan294/K5-L2B-DAY08-2A202603304-BuiThanhTan-AIPrelabel](https://github.com/tantan294/K5-L2B-DAY08-2A202603304-BuiThanhTan-AIPrelabel)
- Commit chốt bài (hash): <<>>
- Máy: Windows 11, NVIDIA RTX 3050 Laptop GPU 4 GB (CUDA)
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 399 / 7 / 29 = 91.7% / 1.6% / 6.7% (mẫu số: 435 box máy); added: 66 = 14.0% (mẫu số: 472 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 27/65 = 41.5% (KTC 95%: 30–54%) / 27/39 = 69.2% (KTC 95%: 54–81%), 4 ảnh gold
- Bắt bẫy: lượt duyệt đầu 8/12 = 66.7% (KTC 95%: 39–86%); sau khi sửa 11/12 = 91.7% (KTC 95%: 65–99%)
- Thời gian (ước lượng, chia từ tổng ~180 phút, không bấm giờ từng bước): gán tay 5 phút/ảnh (20 phút / 4 ảnh); duyệt có pre-label 3 phút/ảnh (72 phút / 24 ảnh)
- Lỗi QC (vòng 3): 20% theo ảnh (1/5, cận trên Wilson 95%: 62%), 1.1% theo đối tượng (1/91, cận trên 6%)
- **Quyết định:** Rework — lý do một dòng: ảnh lỗi 1/5 = 20% > 2% (cận trên 62%), và lượt duyệt đầu chỉ bắt 8/12 bẫy (< 80%), nên phải áp guideline v1.1 cho cả 24 ảnh rồi QC lại với mẫu mới.
