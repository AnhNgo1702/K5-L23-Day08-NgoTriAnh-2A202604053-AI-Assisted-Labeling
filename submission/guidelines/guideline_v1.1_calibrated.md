# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202604053 - Ngô Trí Anh

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện nhỏ: bicycle hay motorbike? | Phân định theo bàn đạp trợ lực và kết cấu: có bàn đạp và dáng xe đạp gán `bicycle`; không có bàn đạp dáng scooter/xe máy gán `motorbike` | img_004.jpg | CAL-02 |
| 2 | Khối xe máy đỗ dày đặc gần camera: từng xe hay ignore_region? | Xe phân biệt rõ nét gán từng chiếc; cụm xe chen chúc bị che khuất chồng chéo quá nặng hoặc kích thước vật < 20 px dùng một vùng `ignore_region` bao quanh | img_024.jpg | CAL-01 |
| 3 | Hàng hoá nhô ra trên xe: box xe có gồm hàng không? | Giỏ hàng, thùng hàng gắn cố định vào xe tính vào box xe; hàng hoá chở rời nhô ra quá 20% chiều dài thân xe thì không bao gồm vào box xe | img_013.jpg | CAL-01 |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_013.jpg | Xe tuk-tuk / ba gác có mui chở người trên đường | YOLO26 gán nhầm thành `car` hoặc `truck`; YOLOE bắt box lỏng | Luôn gán `three_wheeler` cho phương tiện 3 bánh cơ giới hoặc thô sơ; box bao cả mui xe | Mục 2.2: Phân loại three_wheeler có mui xe |
| img_010.jpg | Bẫy cài đổi nhãn người đi bộ thành xe đạp | Pre-label gán nhãn sai `bicycle` cho người đi bộ (box đúng) | Nhận diện và sửa lại `person`; bắt buộc bật bộ lọc checklist rà soát nhãn | Mục 3.1: Quy trình rà soát nhãn độc lập để chống thiên kiến |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt phân định xe điện (CAL-02), quy tắc box xe ba bánh có mui (CAL-01), và quy trình kiểm soát bẫy nhãn sai lớp (CAL-03) | CAL-01, CAL-02, CAL-03 |
