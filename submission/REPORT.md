# Báo cáo Day 08 — 202603304 - Bùi Thanh Tân

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m (`yolo26m.pt`) | `--conf 0.3` mặc định cho cả 6 lớp, không đặt ngưỡng riêng | r01/r02: hạ từ 0.5 xuống 0.25 làm tăng box trên 8 ảnh pilot từ 140 lên 215, bắt thêm được xe và người ở xa (img_007: motorbike 5→11). r05_all + `qc/threshold_sweep.csv`: trên 4 ảnh gold, không lớp nào đạt recall 90% ở ngưỡng ≤ 0.5. Recall tối đa: person 11/15 = 73%, motorbike 7/8 = 88%, car 7/8 = 88%. bicycle và truck có 0 vật gold, bus có 1, nên không chọn ngưỡng riêng được. |
| three_wheeler | YOLOE-26s (`yoloe-26s-seg.pt`), prompt `three_wheeler` | `three_wheeler=0.15` | r04: bắt được tuk-tuk (0.93, img_002) và xích lô (0.92, img_032) mà YOLO26 không có lớp. r06 + sweep: ở 0.15, recall 7/7 = 100%, precision 7/9 = 78%. |

- Lệnh `merge_prelabels.py` đã dùng (YOLOE đặt trước để `three_wheeler` thắng khi chồng box):
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
  Kết quả: 606 box. Có 11 car, 6 truck, 2 bus, 1 bicycle của YOLO26 bị loại vì chồng lên box `three_wheeler`.
- **Phiên bản SAM:** chọn **SAM 2.1-b (`sam2.1_b.pt`)**. Em có so với SAM 1 (`mobile_sam.pt`) trên img_013. SAM 2.1-b cho viền bám thân tuk-tuk tốt hơn, tốc độ chấp nhận được (2.11 s/ảnh ở r05, RTX 3050 4 GB). Nhưng SAM 2.1 cũng dễ lan sang người ngồi trong xe hơn (xem mục 5). MobileSAM nhẹ hơn, phù hợp nếu cần chạy nhanh.
- **Lỗi không sửa được bằng đổi ngưỡng:** YOLO26 không có lớp xe ba bánh nên gán sai lớp với điểm cao. Ví dụ tuk-tuk img_011 → `truck` 0.77, xích lô img_032 → `bicycle` 0.65; ở r02, tăng conf lên 0.5 các box này vẫn còn. Cách sửa: thêm YOLOE với prompt `three_wheeler`, đặt trước YOLO26 khi merge với IoU 0.6, rồi ghi quy tắc "đếm bánh" vào guideline v1.1 (CAL-02).
- **Box điểm cao mà sai:** `truck` 0.77 trên tuk-tuk (img_011), `bicycle` 0.65 trên xích lô (img_032), `three_wheeler` 0.40 trên xe máy kèm người lái (img_002). **Điểm thấp mà đúng:** `three_wheeler` 0.28 cho tuk-tuk bị cắt mép (img_011), `three_wheeler` 0.38 cho xích lô (img_035, #481), các xe máy ở xa có `motorbike` 0.28 (img_007).
- **Giới hạn:** ngưỡng được chọn trên chính 4 ảnh gold (39 đối tượng; three_wheeler chỉ có 7) rồi chấm trên cùng số ảnh đó, nên precision/recall ở mục 2 là con số **lạc quan**. Ngưỡng 0.15 chỉ là ước lượng thô.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 399 | 399/435 = 91.7% (KTC 95%: 89–94%) | 435 box máy |
| edited | 7 | 7/435 = 1.6% | 435 box máy |
| deleted | 29 | 29/435 = 6.7% | 435 box máy |
| added | 66 | 66/472 = 14.0% | 472 đối tượng sau duyệt |

(Đã loại 21 đối tượng quanh các bẫy khỏi bảng. Có 7 ca deleted + added thực chất là đổi lớp, nên số vật máy sót thật ≈ 66 − 7 = 59.)

- **Lớp máy làm tệ nhất:**
  - `three_wheeler`: chỉ 11/25 = 44% box máy được giữ nguyên, 10/25 = 40% phải xoá (KTC 95%: 23–59%). Nguyên nhân là ngưỡng 0.15 thấp, kéo theo nhiều FP như xe máy kèm người lái hay sạp dù (img_002 ở pilot).
  - `truck`: 5/7 box bị xoá, phần lớn thật ra là tuk-tuk hoặc ba gác (img_011).
  - `motorbike`: máy **sót** nhiều nhất, phải vẽ thêm 41 box, chủ yếu là xe ở xa và xe đỗ thành hàng (img_006, img_007).
- **So với gold:** máy đạt precision 27/65 = 41.5% (KTC 95%: 30–54%) và recall 27/39 = 69.2% (KTC 95%: 54–81%). Mẫu số là 39 đối tượng gold trên 4 ảnh.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy ở lượt duyệt đầu:** 8/12 = 66.7% (KTC 95%: 39–86%), **dưới** ngưỡng 80%.
  - Bẫy lọt: missing ở img_003 (người đàn ông trên xe ba gác) và img_031; extra ở img_006 (box `motorbike` ma trên kệ bưu thiếp, giữ nguyên với IoU 1.0); loose ở img_025 (box person lệch khoảng 14 px, chưa kéo lại).
  - Lý do: bỏ bước 1 của checklist. Lẽ ra phải ẩn nhãn máy và đếm người/xe trước, nhưng lại nhìn box có sẵn rồi bấm chấp nhận.
- **Sau khi sửa** (`batch_reviewed.json` export lại): 11/12 = 91.7% (KTC 95%: 65–99%). Con số này **không** đo được độ cẩn thận khi duyệt. Con số dùng để đánh giá vẫn là 8/12.
  - Bẫy img_031 vẫn bị tính lọt là có chủ ý. Box máy gốc chỉ tới ngang hông. Em kéo box tới chân vì chân vẫn nhìn thấy giữa hai bánh xe (guideline 4.1.1–2), nên IoU với box bẫy chỉ còn 0.45 (CAL-05).
## 4. Máy có đáng dùng không?

- **Thời gian (x `time_log.csv`):** gán tay 20 phút / 4 ảnh = **5 phút/ảnh**; duyệt có pre-label 72 phút / 24 ảnh = **3 phút/ảnh**.
  
  - **Thiên lệch:** gán gold trước nên lúc duyệt đã quen cảnh. Ảnh batch lại nhiều vật hơn (472 đối tượng / 24 ảnh ≈ 20 vật/ảnh, so với 39 / 4 ≈ 10 vật/ảnh ở gold).
- **Kết luận:** **có, nhưng chỉ cho person / car / bus.** Với `motorbike` thì chỉ dùng như gợi ý. Với `three_wheeler` phải duyệt kỹ từng box. Căn cứ:
  - 399/435 = 91.7% box máy được giữ nguyên.
  - Máy vẫn sót nhiều xe máy và gán sai lớp xe 3 bánh với điểm cao .
  - Recall trên gold chỉ 27/39 = 69%.
- **Ba quy tắc giảm thiên kiến của **:
  1. **Ẩn nhãn máy, đếm trước:** với mỗi ảnh, ghi số người/xe đếm bằng mắt rồi mới bật box. Kiểm tra bằng tỉ lệ bắt bẫy missing (lượt đầu 1/3) phải đạt 3/3 ở lô sau.
  2. **Lọc theo lớp nghi ngờ:** dùng Filters của CVAT lọc `truck`/`car`/`bicycle`, đếm bánh từng box (CAL-02). Kiểm tra: không còn box `truck`/`car` nào nằm trên xe 3 bánh trong mẫu QC.
  3. **Lọc box trùng trước khi export:** không còn cặp cùng lớp nào có IoU ≥ 0.9 (CAL-04). Kiểm tra bằng script đếm cặp trùng. Ở QC vòng 2 có 3 cặp, vòng 3 còn 0.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013 (tuk-tuk hồng, có tài xế + khách) | SAM 1 (`mobile_sam.pt`) | 3 (2 dương: mái, thân; 1 âm: tài xế) | Không gồm tài xế và khách | Tách tài xế khá sạch, nhưng mặt và bàn chân tài xế vẫn dính mảng hồng | `tuktuk_img013_sam1.jpg` |
| img_013 (cùng vật, so sánh) | SAM 2.1-b (`sam2.1_b.pt`) | 3 (cùng vị trí click) | **Có lẫn** người ngồi trong xe và một phần tài xế, dù có click âm | Lan sang người phía sau song sắt và chân tài xế; mask lốm đốm ở vùng song sắt | `tuktuk_img013_sam2.jpg` |
| img_004 (tuk-tuk xanh dương phía trước) | SAM 2.1-b | 3 (2 dương, 1 âm ở người bên trái) | Không có người; gồm kính chắn gió | Lem nhẹ xuống bóng đổ dưới bánh trước (lốm đốm tối khoảng x≈680–720, y≈550–565) | `tuktuk_img004_sam2.jpg` |
| img_017 (tuk-tuk hồng bên phải) | SAM 2.1-b | 3 (2 dương, 1 âm trên mái) | Gồm ghế và khung trong xe | Mask phủ trọn xe, không lan sang tuk-tuk xanh bên cạnh. Click âm đặt trên mái không có tác dụng (mái vẫn trong mask) | `tuktuk_img017_sam2.jpg` |

- **Đối chiếu guideline mục 4.2:**
  - Quy tắc 4.2.3 (bóng đổ không thuộc vật) giúp nhận ra lỗi ở img_004.
  - Quy tắc 4.2.4 (vùng chạm thuộc vật phía trước) giúp tách tuk-tuk ở img_017.
  - **Còn thiếu:** guideline chưa nói người ngồi *trong* xe ba bánh có bị trừ khỏi mask xe không. Ca img_013 cho thấy SAM 2.1 tự gộp người vào. Câu hỏi 5 về kính xe cũng chưa chốt (img_004 đang tính kính vào mask).
- *(Tuỳ chọn)* Interactor SAM trong CVAT: không thử. Toàn bộ mask làm offline bằng `sam2_points.py`.

## 6. Calibration (Part 6)

- **Số quyết định:** 5 (`qc/calibration_decisions.csv`): 2 `cau_hoi_mo`, 1 `loi_may`, 2 `bay_lot`.
- **Quan trọng nhất là CAL-02** (img_011): xe 3 bánh luôn là `three_wheeler`, khi merge đặt YOLOE trước YOLO26. Đây là lỗi máy có hệ thống và đổi ngưỡng không sửa được.

- **Nguyên nhân gốc:**
  - **Máy** (CAL-02): mô hình COCO không có lớp xe 3 bánh.
  - **Guideline mơ hồ** (CAL-01, CAL-04): chưa có quy tắc cho hàng nhô ra và box chồng nhau.
 

## 7. QC và quyết định (Part 6)

- **Mẫu:** 5 ảnh (0 ảnh `can_review` + 5 ngẫu nhiên, seed 42): img_003, img_006, img_014, img_031, img_035, tổng 91 đối tượng. Người kiểm là em, nên có **hạn chế**: em vừa là người duyệt vừa là người kiểm.
- **QC qua 3 vòng** (phiếu vòng 1 và 2 đã lưu lại, vòng 3 là `qc/qc_review.csv`):

| Vòng | Nhãn | Lỗi theo đối tượng | Lỗi theo ảnh |
|---|---|---|---|
| 1 | `batch_reviewed.json` bản đầu | 9/96 = 9.4% | 4/5 = 80% |
| 2 | Sau khi sửa 4 bẫy lọt | 8/96 = 8.3% | 4/5 = 80% (KTC 95%: 38–96%) |
| 3 | Sau khi sửa theo phiếu vòng 2 | **1/91 = 1.1% (KTC 95%: 0–6%)** | **1/5 = 20% (KTC 95%: 4–62%)** |

  Ngưỡng Release là ảnh lỗi ≤ 2%.
- **Top lỗi** (tính gộp các vòng), mỗi loại một ảnh ví dụ:
  - `thua` (lỗi lặp nhiều nhất ở vòng 1–2, 6/9 rồi 5/8): box thêm tay bị nhân đôi ở img_035 (#470/#459, IoU 1.0), box car ma ở img_006.
  - `bo_sot`: người trên xe ba gác ở img_003.
  - `sai_vi_tri_kich_thuoc`: box xe đạp #413 ở img_031 cắt mất mẹt hàng. Đây là lỗi duy nhất còn lại ở vòng 3.
  - Không gặp `sai_lop`, `sai_thuoc_tinh`, `sai_mask` trong mẫu.
- **Giới hạn mẫu nhỏ:** 1 ảnh lỗi trong 5 ảnh cho cận trên Wilson 95% của tỉ lệ ảnh lỗi thật là **62%**. Với 5 ảnh, kể cả 0/5 lỗi cũng **không chứng minh được** tỉ lệ ≤ 2% (cận trên khi 0/5 vẫn khoảng 43%). Muốn chứng minh ngưỡng 2% cần mẫu lớn hơn rất nhiều.
- **Script kiểm được:** đếm accepted/edited/deleted/added theo IoU, bắt bẫy, precision/recall so với gold, cặp box trùng.
  **Script không kiểm được:** box có ôm sát đúng *phần nhìn thấy* hay không và người cùng bỏ sót (ngoài gold và bẫy thì không có đáp án)

**Quyết định: Rework**, ngày 2026-11-17.

Lý do, đi theo thứ tự Hold → Rework → Release:
- **Rework:**
  - 1/5 = 20% ảnh lỗi, vẫn > 2% (cận trên 62%).
  - Lượt duyệt đầu chỉ bắt 8/12 = 67% bẫy, < 80%. 11/12 là con số sau khi đã biết bẫy nằm đâu, nên không đủ để tin cả lô.
  - Lỗi lặp lại có hệ thống ở vòng 1–2.


## 8. Ghi chú cá nhân
- Sử dụng gemini để tra các định nghĩa cũng như là chụp ảnh lại fix các lỗi biên dịch ở trên terminal chạy lệnh, wsl chạy bị lỗi do lỗi python 3.14 nhờ ai chỉ cách cài lại 3.12 phù hợp với bài lab, cài venv. 
- **Gold có bị nhiễm không:** không. Em gán tay 4 ảnh gold trước khi xem pre-label và không hỏi AI.
- **Máy / GPU / CVAT:** Windows 11, NVIDIA RTX 3050 Laptop 4 GB (CUDA), CVAT v2.81.0 chạy Docker. Vướng mắc khi cài:
  - venv trong repo thiếu numpy, nên các script phải chạy bằng `C:\Python313\python.exe`;
  - mở CSV bằng Excel làm khoá file. `model_log.csv` từng bị lưu hỏng (cắt chữ giữa ký tự UTF-8), đã khôi phục.
- **Lỗi / bug script đã gặp:** `prelabel.py` ghi `run_id` trùng `r05` cho cả lượt SAM và lượt `--split all` trong `work/runs.csv`, nên trong `model_log.csv` em đổi lượt thứ hai thành `r05_all`.
