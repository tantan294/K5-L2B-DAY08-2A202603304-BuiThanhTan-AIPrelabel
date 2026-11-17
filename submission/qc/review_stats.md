# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 435; đối tượng sau duyệt: 472; ngưỡng accepted: IoU >= 0.9
- Đã **loại 21 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 399 | 399/435 = 91.7% | 435 box máy |
| edited (sửa) | 7 | 7/435 = 1.6% | 435 box máy |
| deleted (xoá) | 29 | 29/435 = 6.7% | 435 box máy |
| added (vẽ thêm) | 66 | 66/472 = 14.0% | 472 đối tượng sau duyệt |

Trong số deleted + added, **7 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 59.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 248 | 0 | 3 | 12 |
| bicycle | 6 | 0 | 3 | 1 |
| motorbike | 85 | 3 | 1 | 41 |
| car | 45 | 0 | 6 | 5 |
| bus | 2 | 0 | 1 | 2 |
| truck | 2 | 0 | 5 | 1 |
| three_wheeler | 11 | 4 | 10 | 4 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 39 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 27 | 38 | 12 | 41.5% (KTC 95%: 30–54%) | 69.2% (KTC 95%: 54–81%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 11/12 = 91.7% (KTC 95%: 65–99%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - wrong_class: 3/3
  - missing: 2/3
  - extra: 3/3
  - loose: 3/3
  - lọt: img_031.jpg (missing)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
