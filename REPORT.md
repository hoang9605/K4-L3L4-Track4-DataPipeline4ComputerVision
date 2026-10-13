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
