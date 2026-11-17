# Báo cáo Day 08 — 2A202604053 - Ngô Trí Anh

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person | YOLO26m | 0.4 | Recall đạt 90% (>= 90%), precision 75% trên 20 vật gold (từ threshold_sweep.md). Chạy thử ở r01 (conf 0.25) thấy nhiều box rác và r02 (conf 0.5) bị sót người mờ, ngưỡng 0.4 cân bằng tối ưu giữa precision và recall. |
| bicycle | YOLO26m | 0.2 | Recall đạt 100% (>= 90%), precision 83% trên 5 vật gold. Lớp ít xe nên hạ ngưỡng xuống 0.2 để không bỏ sót xe đạp ở xa. |
| motorbike | YOLO26m | 0.5 | Recall đạt 90% (>= 90%), precision đạt 90% trên 10 vật gold. Mật độ xe máy trên đường phố châu Á rất dày, ngưỡng 0.5 giúp giảm bớt box trùng lặp và box rác (như quan sát ở run r02). |
| car | YOLO26m | 0.3 (mặc định) | Không có ngưỡng nào <= 0.5 đạt recall 90% (tối đa đạt 80% với 10 vật gold); do đó giữ ngưỡng mặc định 0.3 theo khuyến nghị của threshold_sweep. |
| bus | YOLO26m | 0.3 (mặc định) | Chỉ có 1 vật trong tập gold (< 5 vật), cỡ mẫu quá nhỏ không đủ độ tin cậy thống kê để quét ngưỡng riêng, dùng ngưỡng mặc định 0.3. |
| truck | YOLO26m | 0.3 (mặc định) | Có 0 vật trong tập gold (< 5 vật), không thể quét ngưỡng trên tập gold, dùng ngưỡng mặc định 0.3. |
| three_wheeler | YOLOE-26s, prompt: three_wheeler | 0.15 | YOLO26 theo COCO chuẩn không có lớp này. YOLOE với open-vocabulary prompt `three_wheeler` (run r03) phát hiện tốt tuk-tuk; quét ngưỡng đạt recall 100%, precision 67% trên 6 vật gold tại ngưỡng 0.15. |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên):
```bash
python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "person=0.4,bicycle=0.2,motorbike=0.5,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json
```
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: Tôi chọn **SAM 2.1** (trọng số `sam2.1_b.pt`). Lý do: SAM 2.1 có kiến trúc phân đoạn và xử lý đường biên vượt trội hơn SAM 1 ở các chi tiết phức tạp (lan can, bánh xe, gương chiếu hậu của tuk-tuk), hỗ trợ cơ chế prompt đa điểm (positive/negative points) chính xác. Đồng thời SAM 2.1 bản base có dung lượng vừa phải (~100MB), chạy inference trực tiếp ổn định trên máy local mà không đòi hỏi quyền truy cập gated model trên HuggingFace hay cấu hình phần cứng quá lớn như SAM 3.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào?
Lỗi nhầm lẫn nhãn ngữ nghĩa (semantic misclassification), điển hình là xe tuk-tuk / xe ba gác bị YOLO26 gán thành `car` hoặc `truck` (do tập dữ liệu COCO chuẩn không có nhãn `three_wheeler`). Thay đổi ngưỡng `--conf` chỉ lọc hoặc thêm bớt bounding box chứ không thể chuyển đổi tên lớp dự đoán của mô hình. Cách khắc phục: Sử dụng mô hình open-vocabulary (YOLOE) với prompt chuyên biệt `three_wheeler`, sau đó gộp kết quả qua `merge_prelabels.py` (ưu tiên nhãn `three_wheeler` từ YOLOE khi có IoU trùng lặp với box từ YOLO26) và kết hợp người duyệt (human annotator) kiểm tra, sửa nhãn trực tiếp trên công cụ gắn nhãn (CVAT).
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào):
  - Điểm cao mà sai: Bẫy extra cài sẵn ở `img_013.jpg` (fake_box `car` điểm conf cao nhưng vị trí là khoảng trống lề đường); hoặc ở `img_025.jpg` máy nhận diện vùng thùng hàng thành `truck` với conf cao.
  - Điểm thấp mà đúng: Xe tuk-tuk ở góc nghiêng trong `img_013.jpg` và `img_008.jpg` chỉ có độ tin cậy conf ~0.18 - 0.23 từ YOLOE nhưng là xe ba bánh hoàn toàn chính xác.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan:
Vì tập gold chỉ có 4 ảnh (in-sample tuning), việc tìm ngưỡng tối ưu và đánh giá lại trên cùng tập dữ liệu này tạo ra hiện tượng optimistic bias (số đo lạc quan hơn thực tế, khoảng tin cậy 95% rất rộng: Precision 62–84%, Recall 72–92%). Hơn nữa, những lớp hiếm (bus, truck) không có hoặc có quá ít mẫu đại diện trong 4 ảnh gold nên không thể tối ưu hóa ngưỡng tin cậy theo phân phối thực tế.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 386 | 386/435 = 88.7% | 435 box máy |
| edited | 18 | 18/435 = 4.1% | 435 box máy |
| deleted | 26 | 26/435 = 6.0% | 435 box máy |
| added | 47 | 47/451 = 10.4% | 451 đối tượng sau duyệt |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?
Lớp máy thể hiện kém nhất là `three_wheeler` và `truck`. Cụ thể với `three_wheeler`, máy chỉ có 6 box accepted trong khi có tới 6 box phải edited, 12 box bị deleted và 5 box phải vẽ thêm manually (added). Nguyên nhân: mô hình YOLOE dễ bị nhiễu bounding box (box bị lỏng, lệch khung mui xe như ở `img_008.jpg`, `img_023.jpg`) hoặc nhầm lẫn giữa ô tô với xe ba bánh (như bẫy/lỗi ở `img_024.jpg`). Lớp `truck` cũng chỉ có 2 box accepted nhưng bị xóa tới 4 box do nhận nhầm từ thùng xe ba gác hoặc xe bán tải.
- So với gold: precision / recall của **máy**: Precision: 74.6% (KTC 95%: 62–84%) · Recall: 84.6% (KTC 95%: 72–92%) (mẫu số: 52 đối tượng gold).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 11/12 = 91.7% (KTC 95%: 65–99%); loại nào lọt: lọt 1 bẫy loại `wrong_class` tại ảnh `img_010.jpg` (đối tượng người đi bộ `person` bị đổi thành `bicycle`).
- Vì sao (bạn đã bỏ bước nào của checklist)?
Người duyệt đã bỏ qua Bước 1 (Ẩn toàn bộ nhãn máy để quét toàn cảnh độc lập) và Bước 4 (Duyệt theo từng lớp nhãn) trong checklist 4 bước. Khi thấy bounding box ôm rất khít thân người đi bộ, mắt người duyệt bị "neo thị giác" (visual anchoring), chỉ chú ý kiểm tra độ khít của khung viền mà không đọc kỹ tên nhãn hiển thị trên thanh công cụ CVAT, dẫn đến việc chấp nhận nhãn sai `bicycle`.
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt?
  - Nếu tỉ lệ accepted rất cao mà tỉ lệ bắt bẫy thấp (< 80%): Đây là dấu hiệu điển hình của **thiên kiến tự động hóa (automation bias)** và tâm lý "chỉ click Next". Người duyệt tin tưởng máy móc một cách thụ động, không thực sự thẩm định lại nhãn và vị trí box, dẫn đến việc lọt nhiều lỗi nghiêm trọng vào tập dữ liệu cuối.
  - Trường hợp thực tế ở bài này (accepted 88.7% và bắt bẫy 91.7% = 11/12): Người duyệt duy trì được sự hoài nghi lành mạnh đối với AI, phát hiện hầu hết các box thừa, box thiếu và box bị xô lệch (missing: 3/3, extra: 3/3, loose: 3/3), nhưng vẫn có thể mất tập trung ở các ca hoán đổi nhãn tinh vi (wrong_class) nếu không dùng checklist phân loại độc lập.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 5.0 phút/ảnh (20 phút / 4 ảnh gold) vs duyệt có pre-label 3.75 phút/ảnh (90 phút / 24 ảnh batch). Thiên lệch: Khi gán gold 4 ảnh, người làm tuy tập trung cao độ nhưng phải căn chỉnh từng pixel từ trang giấy trắng. Khi duyệt 24 ảnh batch, dù đã quen nhận diện bối cảnh và lớp nhãn nhưng gặp nhiều ảnh giao thông dày đặc (005, 018, 021, 024, 028, 029) và phải rà soát bẫy cẩn thận; dù vậy tốc độ tổng thể vẫn nhanh hơn gán tay 25% (3.75 phút so với 5.0 phút/ảnh).
- Kết luận: CÓ ĐÁNG DÙNG, đặc biệt rất hiệu quả cho các lớp thông dụng có tỉ lệ accepted cao như `person` (242 accepted), `motorbike` (86 accepted), `car` (42 accepted). Tuy nhiên đối với các lớp hiếm/đặc thù như `three_wheeler` hay các cụm xe máy quá dày, AI chỉ mang tính chất gợi ý sơ bộ, bắt buộc phải có con người kiểm tra và hiệu chỉnh kỹ lưỡng.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Quy tắc "Blind First" (Quét mù trước khi xem máy):** Luôn nhấn phím tắt `H` để ẩn toàn bộ annotations trong 10 giây đầu khi mở ảnh mới; tự mắt quét các đối tượng chính và vùng phức tạp trước khi bật nhãn AI lên đối chiếu. Kiểm tra bằng: cam kết thực hiện ở 100% các ảnh trong lượt duyệt đầu.
  2. **Quy tắc "Category Isolator" (Rà soát cô lập theo nhãn):** Sau khi duyệt hình học box, dùng chức năng Filter by label trên CVAT để lọc riêng từng lớp nhạy cảm (`three_wheeler`, `bicycle`, `truck`) để kiểm tra chéo nhãn, ngăn chặn lỗi `wrong_class` như ở `img_010.jpg`. Kiểm tra bằng: số lỗi wrong_class bắt được đạt 100%.
  3. **Quy tắc "Audit 5 Ảnh / 1 Phút Reset":** Sau mỗi 5 ảnh duyệt liên tục, dừng lại 1 phút nghỉ mắt và mở lại ngẫu nhiên 1 ảnh vừa duyệt để kiểm tra lại tỉ lệ accept; nếu ảnh có 100% box accepted mà không có bất kỳ thao tác edit/delete/add nào ở ảnh có trên 15 đối tượng thì bắt buộc phải rà soát lại ảnh đó.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_004.jpg | SAM 2.1 (sam2.1_b.pt) | 3 click (2 pos, 1 neg) | Không gồm người lái (đã tách người lái riêng thành person) | Mask ôm sát phần thân xe tuk-tuk; sau khi thêm điểm âm đã loại bỏ được phần lớn thân người ngồi lái | tuktuk_img004_sam2.jpg |
| img_013.jpg | SAM 1 (mobile_sam.pt) | 3 click (2 pos, 1 neg) | Vẫn còn dính một phần người lái | Đường biên mask khá thô và răng cưa ở bánh xe; 1 click âm ở đầu chưa thể bóc tách hoàn toàn người lái, phần thân và cánh tay người lái vẫn bị dính vào ghế/cabin xe | tuktuk_img013_sam1.jpg |
| img_013.jpg | SAM 2.1 (sam2.1_b.pt) | 3 click (2 pos, 1 neg) | Vẫn còn dính nhẹ phần lưng/tay người lái | Mask bám viền chi tiết sắc nét hơn SAM 1; tuy nhiên 1 điểm negative vẫn chưa thể tách người hoàn toàn do tông màu áo và ghế xe tiệp nhau; nếu tăng thêm nhiều điểm negative thì mask lại bị lẹm khung sau của xe | tuktuk_img013_sam2.jpg |
| img_017.jpg | SAM 2.1 (sam2.1_b.pt) | 3 click (2 pos, 1 neg) | Không gồm người lái; bao trùm toàn bộ khung xe ba gác | Phần bánh xe sau và bóng gầm xe dễ bị dính vào mặt đường; cần click dương chuẩn xác ở tâm bánh xe | tuktuk_img017_sam2.jpg |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu?
  - Quy tắc giúp: Quy tắc 4.2.1 (bám sát đường biên phần nhìn thấy, dung sai <= 3px) và quy tắc 4.2.3 (bóng đổ không thuộc vật, kính xe thuộc vật) giúp định hướng rõ ràng khi đặt điểm positive trên kính và negative ở bóng đường.
  - Quy tắc thiếu: Chưa hướng dẫn cụ thể cách xử lý đối với trường hợp người lái ngồi lọt thỏm bên trong khung hở của xe ba bánh (đặc biệt khi dùng công cụ AI segmenter như SAM - mô hình luôn có xu hướng gom cả xe và người thành một blob liền mạch). Thực tế thử nghiệm cho thấy 1 điểm negative ở đầu không đủ để SAM tách trọn vẹn người lái; nếu thêm nhiều điểm negative thì lại gây lẹm (under-segmentation) vào thân xe. Guideline cần bổ sung hướng dẫn ưu tiên giữ đúng đường biên ngoài của xe và chấp nhận dung sai khi người ngồi lọt trong mui.
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline:
Thử nghiệm chạy offline bằng script `sam2_points.py` cho tốc độ xử lý chủ động và dễ dàng kiểm soát toạ độ điểm click (pos/neg), trực quan hoá ngay diện tích mask và đa giác polygon xuất ra mà không phụ thuộc vào kết nối server của CVAT AI Tools.

## 6. Calibration (Part 6)

- Số quyết định: 3 (`submission/qc/calibration_decisions.csv`); quan trọng nhất: `CAL-01` (Quy định phân loại xe ba bánh / tuk-tuk có mui chở người/hàng luôn gán `three_wheeler`, không gán `car` hay `truck`).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:
  1. Phân định xe hai bánh điện (CAL-02): có bàn đạp trợ lực xếp vào `bicycle`, dáng scooter không bàn đạp xếp vào `motorbike`.
  2. Quy chuẩn bounding box xe ba bánh có mui (CAL-01): box bao trọn cả mui xe và thân xe, tách người lái thành đối tượng `person` riêng.
  3. Quy trình rà soát nhãn độc lập (CAL-03): bắt buộc dùng bộ lọc phân loại từng lớp trên CVAT để phòng chống bẫy đổi nhãn sai lớp (wrong_class).
- Nguyên nhân gốc: Cả hai nguyên nhân: vừa do **guideline v1.0 còn để mở các câu hỏi định nghĩa** (chưa phân định rõ xe đạp điện vs xe máy điện, chưa làm rõ quy tắc mui xe ba bánh), vừa do **thiên kiến tự động hóa khi duyệt nhãn máy** (quá tin vào vị trí box của pre-label mà không đọc lại tên nhãn).

## 7. QC và quyết định (Part 6)

- Tập Release candidate: 24 ảnh batch, 467 đối tượng (`person=264, motorbike=111, car=47, three_wheeler=21, bicycle=9, bus=6, ignore_region=5, truck=4`) lưu tại `submission/release/` (sinh bởi `build_release.py`).
- Mẫu: 8 ảnh (3 ảnh `can_review` + 5 ảnh ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: người kiểm định cũng chính là người đã duyệt batch_review, có thể mang thiên kiến chủ quan).
- Lỗi theo **đối tượng**: 0/147 = 0.0% · theo **ảnh**: 0/8 = 0.0% (ngưỡng Release ≤ 2%).
- Top lỗi: Không ghi nhận lỗi vi phạm quy chuẩn nào trong 8 ảnh mẫu đã chọn sau khi đã được hiệu chỉnh kỹ lưỡng (0 bỏ sót, 0 thừa, 0 sai lớp, 0 sai kích thước).
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): Với mẫu 8 ảnh (0 ảnh lỗi), khoảng tin cậy 95% Wilson của tỉ lệ ảnh lỗi thật có cận trên lên tới **32%**. Do kích thước mẫu kiểm định nhỏ (8/24 ảnh), về mặt thống kê toán học chưa thể chứng minh chắc chắn tỉ lệ lỗi của toàn bộ kho dữ liệu nhỏ hơn ngưỡng khắt khe 2%.
- Script kiểm được / không kiểm được: Script tự động kiểm tra được tính đầy đủ của file, cấu trúc trường dữ liệu, ngưỡng IoU, số lượng box/ảnh, và bắt bẫy. Tuy nhiên, script KHÔNG THỂ tự đánh giá được độ khít mỹ thuật của viền biên đối tượng phức tạp, các ca tranh chấp ranh giới ngữ nghĩa (ví dụ người ngồi lấp ló sau rèm tuk-tuk) hay các lỗi mà cả máy lẫn người cùng bỏ sót do thiên kiến chung.

**Quyết định: Release**, ngày 2026-11-17.
Lý do (đi theo Hold → Rework → Release):
1. Kiểm tra Hold: Không còn câu hỏi mở guideline nào chưa giải quyết (đã chốt toàn bộ trong v1.1), không có ảnh bị lỗi dữ liệu hay hỏng file -> Không cần Hold.
2. Kiểm tra Rework: Tỉ lệ ảnh lỗi trong mẫu QC là 0.0% (đạt ngưỡng <= 2%), tỉ lệ bắt bẫy đạt 91.7% (11/12 >= 80%), không có lỗi hệ thống lặp lại -> Không cần Rework.
3. Kết luận: Bộ dữ liệu đáp ứng đầy đủ tiêu chí nghiệm thu của quy trình kiểm định chất lượng -> Đạt điều kiện **Release**.

Nếu Rework / Hold, việc cần làm tiếp: Hiện tại không có yêu cầu Rework / Hold. Nếu trong tương lai mở rộng quy mô dữ liệu, cần bổ sung thêm annotator thứ hai độc lập để đo chỉ số Cohen's Kappa / Inter-Annotator Agreement.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài**: Sử dụng mô hình YOLO26m, YOLOE-26s, SAM 2.1 (`sam2.1_b.pt`) để sinh pre-label; sử dụng trợ lý code để hỗ trợ chạy phân tích script và đối soát báo cáo; toàn bộ nhãn và số liệu đều được kiểm tra đối chiếu thực tế trên file xuất COCO và CVAT.
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? Không; 4 ảnh gold được gán tay hoàn toàn thủ công trên CVAT trước khi chạy bất kỳ mô hình AI nào ở Part 2, đảm bảo tính khách quan làm chuẩn đo (ground truth).
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, CPU (Intel), CVAT chạy local qua Docker v2.81.0. Do chạy CPU nên mô hình SAM mất ~8.9 giây/ảnh, tuy nhiên đã tối ưu hóa chạy ổn định trên các tập mẫu.
- Dùng cache `data/cache/` thay vì tự chạy? Không; tự chạy trực tiếp các lệnh script `prelabel.py`, `threshold_sweep.py`, `merge_prelabels.py`, `sam2_points.py` trên môi trường máy cá nhân.
- Lỗi / bug script đã gặp (nếu có): Chú ý khi chạy PowerShell trên Windows cần đặt các tham số chuỗi có dấu phẩy hoặc dấu bằng trong dấu nháy kép (ví dụ `--thr "person=0.4,..."`) để tránh PowerShell tự tách đối số.
- Một điều bạn làm khác lần sau: Ở các bài toán sau, sẽ luôn bật bộ lọc checklist theo từng nhóm lớp nhãn ngay từ đầu lượt duyệt batch để triệt tiêu hoàn toàn khả năng lọt bẫy đổi nhãn sai lớp (`wrong_class`).
