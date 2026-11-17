# Label contract — Phố châu Á — BỔ SUNG v1.1 — 202603304 - Bùi Thanh Tân

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 3 | **Hàng nhô ra**: xe đạp / ba gác chở rổ, mẹt nhô xa, box xe có gồm hàng không? Mask thì sao? | Hàng đặt cố định trên xe (giỏ, mẹt, rổ buộc trên baga) nhô ra **≤ 20% chiều dài xe** thì **tính vào** box xe. Phần nhô **> 20%** thì box dừng ở thân xe. Mask dùng cùng ngưỡng. Ở img_031, box xe đạp phải kéo sang trái tới mép mẹt cà chua (x≈140), không được chỉ tính giỏ bên phải. | `img_031` (#413) | CAL-01 |
| 4 | **Người ngồi trong xe ba bánh có mui che**: ngưỡng "còn thấy ≥ 20%" có hợp lý không? | Giữ ngưỡng 20% nhưng đổi sang tiêu chí dễ đo bằng mắt: **thấy rõ đầu + vai là đủ** để gán `person`. Mỗi người một box, kể cả khi bị người ngồi trước che một phần. | `img_003` (người đàn ông #1, người phụ nữ đội nón #20 trên xe ba gác) | CAL-03 |
| 6 | **Box chồng nhau quá nhiều**: gộp hay giữ? | **Một vật = một box.** Hai box cùng lớp có IoU ≥ 0.9 là box trùng: giữ box khít hơn, xoá box kia. IoU 0.5–0.9 thì mở ảnh kiểm từng box, nếu là hai xe đỗ liền kề khác nhau thì giữ cả hai. Áp dụng cho cả ảnh ngày lẫn đêm. | `img_035` (#458/#471, #459/#470 trùng khít IoU 1.0), `img_006` (#68/#78 IoU 0.64 là hai xe khác nhau, giữ) | CAL-04 |

**Chưa chốt (giữ cho v1.2):**
- Câu 1 (xe đạp điện): lô này chưa có ảnh ví dụ rõ.
- Câu 2 (khối xe máy đỗ dày gần camera): mới thấy ở img_006, chưa đủ ca để đặt ngưỡng.
- Câu 5 (kính xe trong mask): chưa làm đủ mask để quyết.
- Câu 7 (conf tối thiểu để chấp nhận không sửa): theo `threshold_sweep`, chỉ `three_wheeler` có ngưỡng đáng tin (0.15, 7 vật gold). Các lớp khác không đạt recall 90% ở mọi ngưỡng ≤ 0.5, nên **không** đặt mức chấp nhận tự động.

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_011` (cả `img_004`, `img_032`) | Tuk-tuk, xích lô trên phố | **YOLO26-m** không có lớp xe ba bánh nên gán sai lớp với conf cao: tuk-tuk → `truck` 0.77 (img_011), `car` 0.37 (img_004); xích lô → `bicycle` 0.65 (img_032). Tăng `--conf` không sửa được. | Đổi tất cả thành `three_wheeler`. Khi merge, đặt YOLOE (`three_wheeler`) trước YOLO26 để box của YOLOE thắng. (CAL-02) | **4.3**: đếm bánh trước khi tin lớp máy, 3 bánh → `three_wheeler`. Khi duyệt, lọc riêng `car`/`truck`/`bicycle` để soát lại. |
| `img_003` | Hai người ngồi trên xe ba gác, sau một em bé | Bẫy `missing` xoá box người đàn ông; lượt duyệt đầu **không vẽ lại** (bẫy lọt). Người phụ nữ đội nón phía sau cũng bị sót tới vòng QC 3. | Mỗi người còn thấy rõ đầu + vai là một `person` riêng. (CAL-03) | **7 / checklist**: ẩn nhãn máy, đếm người trên từng xe trước, rồi mới đối chiếu box. |
| `img_031` | Người phụ nữ dắt xe đạp chở hàng, chân lộ ra giữa hai bánh | Box máy gốc chỉ tới ngang hông. Box đã sửa tới chân thì có IoU 0.45 với box bẫy nên bị tính lọt (bắt bẫy 11/12). | **Giữ box tới chân** theo 4.1.1–2 (ôm sát phần nhìn thấy). Không sửa box cho khớp bẫy. (CAL-05) | **4.1.2** thêm ví dụ: chân nhìn thấy qua khoảng hở bánh xe vẫn thuộc box `person`. |
| `img_035`, `img_006` | Box xe máy trùng, box xe máy đè lên lưng người lái | Có box thêm tay bị nhân đôi (IoU 1.0) và một box `motorbike` thừa đè lên người lái #476 (xe đó đã có box #478). `car` 0.55 ma ở cuối hẻm img_006. | Xoá box trùng và box ma. (CAL-04) | **4.1.8 (mới)**: một vật một box; trước khi export, lọc các cặp cùng lớp IoU ≥ 0.9. |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở 3 (hàng nhô ≤ 20% thuộc box xe), 4 (đầu + vai đủ để gán người trong xe ba bánh), 6 (một vật một box, IoU ≥ 0.9 là trùng). Thêm quy tắc 4.3 (đếm bánh, xe 3 bánh luôn là `three_wheeler`), 4.1.8 (lọc box trùng), ví dụ 4.1.2 (chân lộ qua bánh xe), bước checklist "đếm người trên từng xe". Câu 1, 2, 5, 7 chưa chốt. | CAL-01 (img_031), CAL-02 (img_011), CAL-03 (img_003), CAL-04 (img_035), CAL-05 (img_031) |
