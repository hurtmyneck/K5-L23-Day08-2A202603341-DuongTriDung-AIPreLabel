# Báo cáo Day 08 — 2A202603341 - Dương Trí Dũng
## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person | YOLO26-s | 0.4 | Gold: recall 93%, precision 58% trên 15 vật (`qc/threshold_sweep.md`). r05 chạy cả 36 ảnh ở conf 0.05 để quét ngưỡng. |
| motorbike | YOLO26-s | 0.3 | Gold: recall 90%, precision 82% trên 10 vật. |
| car | YOLO26-s | 0.45 | Gold: recall 100%, precision 73% trên 8 vật. |
| bicycle, bus, truck | YOLO26-s | mặc định 0.3 | Ít hơn 5 vật trong 4 ảnh gold (bicycle 0, bus 1, truck 0): không chọn ngưỡng riêng theo hướng dẫn Part 3. |
| three_wheeler | YOLOE-26s, prompt `three_wheeler` | 0.15 | YOLO26 (COCO) không có lớp này nên gọi tuk-tuk là car/truck; YOLOE gõ chữ ra box (r03, r06). Gold: recall 100%, precision 78% trên 7 vật. |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):`python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "person=0.4,motorbike=0.3,car=0.45,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM đã chọn:** SAM 2.1 (`sam2.1_b.pt`). 
- **Lỗi không sửa được bằng đổi ngưỡng:** tuk-tuk bị YOLO26 gọi là `car`/`truck` (lỗi phân loại của model closed-set COCO, không phải điểm thấp). Sửa bằng cách dùng YOLOE với prompt riêng cho `three_wheeler`, rồi gộp hai nguồn bằng `merge_prelabels`.
- Box điểm cao mà sai / điểm thấp mà đúng: ví dụ box `three_wheeler` ở img_003 id 23 (conf 0.90) đúng, nhưng box id 24 (conf 0.80) là box phụ chồng lên cùng xe và bị xoá, tức điểm 0.80 vẫn sai. Ngược lại img_013 id 143 `motorbike` conf 0.31 đúng chỗ nhưng ôm gộp hai xe, phải tách.
- Giới hạn: ngưỡng chọn trên chính gold (4 ảnh) nên precision / recall của máy trên gold là lạc quan (chọn và chấm trên cùng dữ liệu). Lớp ít vật cho kết luận yếu.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 343 | 85,8% | 400 box máy |
| edited | 16 | 4,0% | 400 box máy |
| deleted | 39 | 9,8% | 400 box máy |
| added | 55 | 13,2% | 416 đối tượng sau duyệt |

- **Lớp máy tệ nhất: `three_wheeler`.** Trên 39 box máy: accepted 6, edited 9, deleted 24. Phần lớn box bị xoá là box phụ chồng lên cùng một xe (img_003, img_017, img_023, img_008) hoặc vật không phải xe ba bánh (giỏ hàng, xe đẩy ở img_014, img_009). Lớp khác: person added 18, motorbike added 18, motorbike deleted 8, truck deleted 3.
- So với gold: precision của **máy** 63,9% (39 TP / 61 box dự đoán, KTC 95%: 51–75%) và recall 95,1% (39 / 41 đối tượng gold, KTC 84–99%). Mẫu số recall: 41 đối tượng gold trên 4 ảnh. Số này lạc quan vì ngưỡng chọn trên chính gold.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **10/12 = 83,3%** (KTC 95%: 55–95%, ngưỡng đạt 80%). Theo loại: extra 3/3, missing 3/3, wrong_class 3/3 (img_003, img_008, img_027: `person` bị gán `bicycle`), loose 1/3.
- Loại lọt: `loose`, ở img_005 (id 65 `person`, box bị kéo lệch) và img_028. Vì sao: đây là hai ảnh xe máy dày (nhóm ảnh đông mà hướng dẫn đã cảnh báo), nơi lệch vài chục pixel khó thấy bằng mắt qua ảnh render; bước 4 của checklist (box có ôm sát vật không) là bước làm kém nhất.
- Accepted đạt 85,8% mà bắt bẫy vẫn đạt 83,3% cho thấy khi duyệt đã không chỉ bấm Accept; nhưng chỗ lọt tập trung ở lỗi box lỏng, không phải lỗi rõ như thiếu hoặc thừa. 
## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 3 phút/ảnh so với duyệt 0.5 phút/ảnh. 
- Kết luận (đề xuất): **đáng dùng cho `person`, `motorbike`, `car`** (accepted cao, recall gold ≥ 90%); **không đáng tin cho `three_wheeler`** (deleted 24/39 box) nếu không có bước duyệt. Dựa trên: accepted 85,8%, recall gold 95,1%, deleted three_wheeler 24/39.
- **Ba quy tắc giảm thiên kiến** (mỗi quy tắc kiểm tra được):
  1. Với mỗi ảnh, đếm và ghi số vật theo lớp **trước** khi bật nhãn máy; kiểm bằng cách so số đếm với số đối tượng sau duyệt, lệch > 20% phải duyệt lại ảnh đó.
  2. Với lớp `three_wheeler`, mỗi xe một box: chạy kiểm tra các cặp box cùng lớp có IoU > 0.5 trong một ảnh; kết quả phải bằng 0.
  3. Với ảnh dày xe máy (> 20 box), thêm một lượt riêng chỉ kiểm độ ôm sát của box; kiểm bằng tỉ lệ bắt bẫy `loose` trên 3 bẫy (mục tiêu ≥ 2/3).
  

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013 | SAM 2.1 (`sam2.1_b.pt`) | 3 (2 dương, 1 âm) | Gồm cả người lái và ghế; click âm vào người lái không loại được | Người lái lẫn vào mask vì ngồi trong khung xe và cùng màu / cùng vùng với thân xe; có vài chấm thủng | `tuktuk_img013.jpg` |
| img_023 | SAM 2.1 | 3 (2 dương, 1 âm) | Không có người lái trong mask; tách đúng chiếc tuk-tuk trái khỏi chiếc thứ hai sát bên | Dải ruy-băng hồng treo trên mui bị lẫn vào mask | `tuktuk_img023.jpg` |
| img_009 | SAM 2.1 | 3 (2 dương, 1 âm) | Gồm cả xe máy và hàng chuối nhô ra bên phải | Hàng nhô quá ~20% chiều dài xe nằm trong mask, mâu thuẫn với quy tắc box ở guideline 4.1.5 | `loadedbike_img009.jpg` |
| img_013 (so sánh) | MobileSAM (`mobile_sam.pt`) | 3 | Gồm người lái | Bỏ sót nhiều vùng trong khoang xe; mask 22,7% so với 44,7% của SAM 2.1 | `tuktuk_img013_mobilesam.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc "polygon bám biên phần nhìn thấy" giúp ở img_023. Quy tắc còn thiếu: chưa nói mask xe có loại người lái ngồi trong khung không (img_013), và chưa nói hàng nhô thì mask xử lý thế nào (img_009). Hàng nhô đã vào v1.1 (CAL-03); phần người lái ngồi trong khung xe chưa có trong v1.1
- Quy tắc / lỗi rút ra: một click âm có thể bị bỏ qua nếu vùng đó nằm trong cùng đối tượng liền khối với vật chính; cần thêm click dương vào vùng cần giữ và click âm gần biên hơn, hoặc dùng `--box`.

## 6. Calibration (Part 6)

- Số quyết định: **5** (`qc/calibration_decisions.csv`): CAL-01, 02, 05 là `loi_may`; CAL-03, 04 là `cau_hoi_mo`. Quan trọng nhất: CAL-01 (một xe một box `three_wheeler`, vì `three_wheeler` là lớp bị xoá nhiều nhất, 24 box).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: một xe một box; van / minivan / minibus nhỏ thuộc `car` và đếm bánh để phân biệt `three_wheeler`; xích lô nhìn từ sau là một `three_wheeler` (không gán `bicycle` cho phần xe đạp); chốt câu hỏi mở 2 (xe đông gần camera: vẽ từng xe nếu cao ≥ 20 px) và 3 (hàng nhô không tính vào xe). 5 câu hỏi mở còn lại chưa chốt vì chưa đủ ca.
- Nguyên nhân gốc: CAL-01, 02, 05 chủ yếu do **máy** (YOLOE sinh box chồng, closed-set gọi sai lớp); CAL-03, 04 do **guideline mơ hồ** (câu hỏi mở của v1.0).

## 7. QC và quyết định (Part 6)

- Mẫu: **8** ảnh (3 ảnh `can_review` + 5 ngẫu nhiên, seed 42), 149 đối tượng. Người kiểm: **AI (Claude)**, không phải bạn. **Hạn chế:** kiểm bằng mắt trên ảnh render, không kiểm được vật nhỏ < 20 px ở ảnh đông (img_018, img_024) hay đối chiếu từng pixel; người duyệt và người kiểm cùng là AI nên không độc lập.
- Lỗi theo **đối tượng**: **1/149 = 0,7%** · theo **ảnh**: **1/8 = 12,5%** (ngưỡng Release ≤ 2%).
- Top lỗi: `bo_sot` 1 (img_003: người ngồi sau xe máy #8 chưa có box, nhưng bị che nhiều nên không chắc).
- **Giới hạn mẫu nhỏ:** 1 ảnh lỗi trong 8 ảnh có cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **47%** (`qc/qc_report_auto.md`). Ngưỡng 2% không thể chứng minh bằng mẫu cỡ này, kể cả khi 0 lỗi.
- Script kiểm được: tên file, lớp, box trong ảnh, đủ ảnh, các tỉ lệ. Không kiểm được: box có ôm sát vật thật không, lớp có đúng thực tế không, vật mà cả người duyệt lẫn máy cùng bỏ sót, việc có theo checklist thật hay không.

**Quyết định: Rework**:  Lý do: guideline đã đủ rõ để làm tiếp (không Hold); ảnh có lỗi 12,5% (1/8) vượt ngưỡng 2%, và với mẫu 8 ảnh không thể chứng minh ngưỡng đạt, nên không thể Release.
Việc cần làm tiếp: người thật kiểm lại ít nhất các ảnh đông và ảnh ca khó (img_005, img_018, img_024, img_028) bằng CVAT, bổ sung box người ngồi sau xe ở img_003, tăng mẫu QC (`qc_sample.py --force` với mẫu lớn hơn), rồi chạy lại `build_release.py`.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** Claude Code.
- Gold có bị nhiễm không? không
- Máy / GPU / CVAT dùng: RTX-3050 4GB
- Dùng cache `data/cache/` thay vì tự chạy? tự chạy 36/36 ảnh
- Lỗi / bug script đã gặp: `import PIL.Image` lỗi trong conda base, nên `qc_sample.py` phải chạy bằng venv khác.
