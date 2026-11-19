# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603341 - Dương Trí Dũng

> Chỉ ghi phần thay đổi so với v1.0. Mỗi ca khó có ảnh ví dụ (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày đặc gần camera: từng xe hay `ignore_region`? | Vật rõ, cao ≥ 20 px thì vẽ từng xe và từng người, kể cả chồng nhau. Chỉ cụm ≥ 5 vật cao < 20 px hoặc không tách được mới dùng một `ignore_region`. | img_005 (gần camera, vẽ từng xe), img_026 (dãy xe phía xa, dùng `ignore_region`) | CAL-04 |
| 3 | Hàng nhô ra: box và mask xe có gồm hàng không? | Không gồm hàng nhô quá ~20% chiều dài xe; hàng gắn cố định trong khung (giỏ, bao buộc trước yên) vẫn tính vào xe. Với mask: click âm vào hàng nhô. | img_009 | CAL-03 |
| 1, 4, 5, 6, 7 | Xe đạp điện, người trong xe có mui che, kính xe, box chồng ở ảnh đêm, ngưỡng conf tối thiểu | **Chưa chốt** ở bản này: lô 24 ảnh chưa có đủ ca để quyết định, giữ nguyên v1.0. | — | — |

## 2. Ca khó mới (từ lỗi của máy, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_003, img_017, img_023, img_008 | Một xe ba bánh | Vẽ 2-3 box `three_wheeler` chồng nhau (YOLOE) | Giữ một box ôm cả xe + mui / thùng; xoá box phụ | Mục 2: mỗi xe một box, không box con cho mui, thùng, bánh (CAL-01) |
| img_003, img_018, img_024, img_034 | Xe ở xa: van, minibus, xe tải nhỏ | Gán `three_wheeler` hoặc `truck` (YOLOE / YOLO26) | Đếm bánh và nhìn hình dáng: van / minivan / minibus nhỏ → `car`; xe khách lớn → `bus`; xe tải có thùng riêng → `truck`; không chắc thì tick `can_review` | Mục 2: thêm van, minivan vào `car`; mục 4.3: đếm bánh (CAL-02) |
| img_010, img_027, img_035 | Xích lô nhìn từ sau / xa | Cắt đôi thân xe, gán `bicycle` cho phần xe đạp, gọi là `motorbike` | Một `three_wheeler` gồm cả bộ phận xe đạp; người đạp là `person` riêng | Mục 5: bổ sung ca xích lô nhìn từ sau / xa (CAL-05) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi 2 và 3; thêm quy tắc một xe một box; van / minivan thuộc `car`; xích lô nhìn từ sau; còn 5 câu hỏi mở chưa chốt | CAL-01 đến CAL-05 |
