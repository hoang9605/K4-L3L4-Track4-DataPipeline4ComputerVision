# REPORT — Lab Day 9

| Thông tin | Giá trị |
|---|---|
| Họ và tên | NguyenHaiHoang |
| MSSV (đúng bằng LAB_STUDENT_ID trong .env) | 2A202602489 |
| GitHub | hoang9605 |
| Trip dùng cho bài tập sập và chạy lại | trip-07304 |

MSSV có chữ nên chọn trip bằng **5 số cuối**, theo chỉ dẫn của sinh viên: `int("02489") % 26 = 19`; phần tử chỉ số 19 trong `list-trips` là `trip-07304`. MSSV đầy đủ `2A202602489` được giữ nguyên khi chia dataset.

## Kết quả cuối

- `dataset_version` của bạn: `d58c7e2b9dfa`.
- Số khung hình train / val / test: **740 / 113 / 166**, tổng **1019**.
- Số trip train / val / test: **18 / 4 / 4**.
- Pipeline thủ công: 5 bước đạt; **27/27 checks passed**. Unit test: **9/9 đạt**.
- Báo cáo được hỗ trợ soạn bằng Codex; số liệu lấy từ lần chạy thực tế trên máy và các file bằng chứng trong `report/evidence/`.

## Bước 1 · Bronze

### Câu 1

Nguồn A có `ts_utc` dạng ISO UTC, `speed_mps` theo m/s, `heading_deg`, `lat/lon/alt_m`, `acc_x/acc_y` và `gas/brake`. Nguồn B đổi tên thành `timestamp_ms` dạng epoch ms, `spd_kmh` theo km/h, `hdg`, `latitude/longitude/altitude`, `ax/ay` và `throttle_pct/brake_pct` theo phần trăm. Đồng hồ B còn đi trước video 1500 ms, còn A không lệch.

Bronze giữ riêng từng nguồn và lưu cột dạng chữ để bảo toàn dữ liệu gốc, kể cả ô trống hoặc giá trị lỗi; việc chuẩn hóa diễn ra ở Silver để có thể truy vết và xử lý lại. Lần chạy nhận 3021 dòng A, 3234 dòng B, 26 trip và 2 video. Ingest A lần hai báo `already present`, vì nhận diện nội dung bằng sha256 nên không tăng số dòng.

## Bước 2 · Silver telemetry

### Câu 2

Staging đổi tên cột của B về cùng schema A, chia `spd_kmh` cho 3,6, chia bàn đạp phần trăm cho 100, đổi epoch ms sang timestamp UTC sau khi trừ `telemetry_clock_offset_ms = 1500`. A dùng timestamp ISO UTC và tốc độ m/s sẵn. B bắt đầu ở **2025-10-08 14:30:00.009 UTC**, chỉ cách thời điểm video 9 ms; A bắt đầu **2025-10-07 08:00:00.014 UTC**. Tốc độ tối đa làm tròn của A/B là **71/100 km/h**.

| Drive | Trùng | Thiếu tốc độ | Tốc độ ngoài miền | Tổng cách ly | Dòng sạch |
|---|---:|---:|---:|---:|---:|
| drive_a | 20 | 30 | 3 | 53 | 2968 |
| drive_b | 20 | 30 | 3 | 53 | 3181 |

Đối chiếu: **2968 + 53 = 3021**, **3181 + 53 = 3234**; tổng sạch 6149, cách ly 106. Lưu vào `telemetry_rejects` cùng lý do thay vì xóa giúp kiểm toán chất lượng, truy ngược nguồn và xử lý lại khi thay luật.

### Câu 3

`max_gap_ms` có `severity: warn` vì mất mẫu cảm biến là lỗi đã biết: cần cảnh báo, còn bước join đánh dấu khung mất đồng bộ. `in_range` mặc định là `error`, vì tốc độ ngoài 0–70 m/s là dữ liệu không hợp lệ và phải chặn trước Gold. Trong `break_gate.json`, nới `max_speed_mps` lên 1000 khiến test **in_range_silver_telemetry_speed_mps__70__0** bắt **6 dòng**; kết quả **PASS=5 WARN=1 ERROR=1 SKIP=0**. Khôi phục luật cho **PASS=6 WARN=1 ERROR=0 SKIP=0**. Luật làm sạch và cổng kiểm tra độc lập giúp bắt việc vô tình nới luật, không chỉ xác nhận lệnh biến đổi đã chạy.

## Bước 3 · Silver frames

### Câu 4

Thí nghiệm trên `trip-07304` trong `idempotency.json`: toàn bảng có **1246 dòng** trước và sau hai lần chạy lại; fingerprint đều **6c7480dfde7e**. Trip có **34 khung**, mô phỏng append hai lần tạo **68 dòng**, tức 2 lần dữ liệu của trip. Pipeline ghi Delta overwrite với predicate `trip_id` nên thay đúng partition, các trip khác không bị nhân đôi.

Delta công bố phiên bản mới bằng commit nguyên tử: người đọc thấy snapshot cũ hoặc mới, thay vì khoảng trống do xóa trước rồi ghi lại. Nếu lỗi trước commit, snapshot trước vẫn dùng được; file chưa được tham chiếu có thể tồn tại nhưng không thành dòng đã commit. Trong mã hiện tại, lỗi mô phỏng xảy ra ở đầu `extract_trip`, trước khi trích xuất/ghi bảng; bài tập này kiểm chứng retry, không chứng minh một crash thật giữa commit.

### Câu 5

Khung đen có độ sáng dưới ngưỡng `dark_luma`, gần như không có tín hiệu; khung tối vẫn có nội dung nhưng độ sáng nằm giữa `dark_luma` và `dark_soft`. Pipeline loại khung đen và gắn cờ khung tối, giữ lại nếu không vi phạm cờ khác. A có **6 đen, 9 mờ, 4 cờ tối, 149 gần trùng, 7 mất đồng bộ**; B có **4 đen, 0 mờ, 61 cờ tối, 53 gần trùng, 7 mất đồng bộ**. Các cờ có thể chồng nhau; số cờ tối không đồng nghĩa mọi khung đó đều vào Gold.

| dHash khác tối đa | Khung trùng A | Khung trùng B |
|---|---:|---:|
| 0 | 61 | 8 |
| 2 | 149 | 53 |
| 6 | 363 | 244 |

Chọn ngưỡng 2 như lab: ngưỡng 0 bỏ sót ảnh gần giống, ngưỡng 6 loại tới 363/602 khung A và 244/644 khung B, có nguy cơ mất cảnh hữu ích. Xe chậm tạo ảnh kế tiếp tương tự dù camera bình thường, nên cần xem ảnh và tốc độ trước khi kết luận đứng hình. Giữ cảnh tối giúp mô hình học đường hầm/bóng râm, nhưng cần xem xét độ hữu ích và nhiễu thay vì chỉ dựa vào độ sáng. Join event time có độ lệch trung bình A/B **66/73 ms**, tối đa **2884/3112 ms** ở đoạn mất cảm biến. Unit test xử lý ảnh: **9/9 đạt**.
