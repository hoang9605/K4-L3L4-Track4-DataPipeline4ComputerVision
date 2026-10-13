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

## Bước 4 · Gold

### Câu 6

`leakage.json` ghi **1019 khung thuộc 26 trip**. Chia ngẫu nhiên theo frame làm **26 trip** xuất hiện ở nhiều split, cả **26** có khung ở train và test; chia theo trip làm **0 trip** bị chia cắt. Khung liền nhau gần như cùng cảnh, nên mô hình có thể ghi nhớ nền/đoạn đường và đạt điểm test cao dù chưa tổng quát được sang chuyến mới. Chia theo trip giữ cả cảnh trong một split để giảm rò rỉ này, nhưng vẫn cần kiểm tra các chuyến khác nhau có lặp tuyến đường hay không.

### Câu 7

Phiên bản là **d58c7e2b9dfa**, được tính từ sha256 của danh sách `frame_id|split` sắp theo frame_id, lấy 12 ký tự đầu. Nó không phụ thuộc giờ chạy hoặc số commit Delta. Lần chạy thủ công và DAG từ lake trống tạo cùng phiên bản, cùng **740/113/166** khung train/val/test. MSSV đầy đủ được trộn vào thứ tự md5 để chia trip; đổi MSSV có thể đổi trip ở các split và vì thế đổi phiên bản. Thí nghiệm đổi MSSV chưa được thực hiện; giải thích này dựa trên mã chia split, không phải số liệu giả lập.

Phiên bản này chỉ băm cặp frame_id/split, **không băm toàn bộ pixel hay mọi cột telemetry**; vì vậy muốn xác nhận đầy đủ dữ liệu cần xem thêm manifest, sha256 đầu vào và lineage. Hai lần chạy lại có thể giữ dataset_version nhưng có thời điểm tạo và phiên bản bảng upstream khác nhau.

### Câu 8

Manifest của lần DAG đầu tiên được lưu trong `report/evidence/dataset_manifest.json`: `bronze.raw_telemetry_a=0`, `bronze.raw_telemetry_b=0`, `bronze.trips_raw=0`, `bronze.video_files=1`, `silver.frames=25`, `silver.telemetry=0`, `silver.frames_flagged=0`, `silver.frame_telemetry=0`. Video A có sha256 `636ba1a98dfc5ba665d1c935e1d2064e69f4df360ff9ffbd15795957d5e7e9aa`; B là `bfa62703f645ff77a8578ae3fc74a00881140d38a404d43c8e2af265bbee9fdc`.

Gold có **436 khung A, 583 khung B**, bỏ `is_black`, `is_blurry`, `is_near_dup`, `not sync_ok`; split theo trip, phân tầng theo drive. Sau 3 tháng, cần giữ video/CSV đúng checksum, snapshot Delta nêu trên nếu còn, mã và cấu hình ngưỡng/split, MSSV `2A202602489`, manifest cùng danh sách frame/split. Lineage cho biết phiên bản nguồn nhưng không tự giữ lại dữ liệu nếu lịch sử đã bị xóa/vacuum. Các lần chạy mới có thể cùng dataset_version dù số phiên bản Delta khác.

## Bước 5 · Điều phối và phục vụ

### Câu 9

Sau reset từ lake trống, lần chạy `lab_recovery_02489` có đúng **1 task đỏ**: `silver_frames`, map index **19**, trip **trip-07304**; **25** mapped task còn lại thành công. **7** task `upstream_failed` gồm `dbt_run_silver`, `dbt_test_silver`, `dbt_run_gold`, `dbt_test_gold`, `publish_lake`, `export_dataset`, `publish_warehouse`. Log ghi `RuntimeError: simulated crash while extracting trip-07304; run it again`.

Trong UI, tắt Downstream và clear **chỉ map index 19** (Affected Tasks: 1). Task chạy lại thành công; snapshot `airflow_after_clear_one.json` cho **30 success + 7 upstream_failed**. Airflow 2.10.5 không tự bỏ trạng thái upstream_failed đã kết thúc, nên tiếp tục clear `dbt_run_silver` cùng downstream, **7 task**, giữ nguyên 25 trip đã thành công. Snapshot `airflow_recovered_tasks.json` cho **37/37 success**. Tổng phục hồi lần đầu: 1 task trích xuất chạy lại; 7 downstream chạy lần đầu sau khi được mở khóa, không phải trích xuất lại cả 26 trip.

Sau phục hồi, verify **27/27**, Silver có **1246** frame_id duy nhất, Gold **1019** khung và dataset_version vẫn **d58c7e2b9dfa**, không nhân đôi. Bằng chứng gồm `airflow_clear_one_task.png`, `airflow_clear_downstream.png`, `airflow_graph.png`, `airflow_recovered.png` và các snapshot JSON. Để chụp rõ tên trip, mô phỏng lỗi lại trên riêng task 19 sau khi lần phục hồi đầu đã xong; ảnh `airflow_crash_log.png` và log gốc `airflow_crash_log.txt` là **attempt 3** của lần bổ sung này, rồi task được clear riêng để phục hồi ở attempt 4. Vì 7 downstream đã thành công, lần bổ sung không có 7 upstream_failed như lần đầu. Các lần thử được phân biệt rõ, không ghép ảnh attempt 3 thành log của attempt 1.

### Câu 10

Ở mức 500 xe/ngày, ba nút thắt dự kiến (chưa benchmark ở quy mô này) là: **decode video**, vì FFmpeg dùng CPU và mỗi trip ghi ảnh riêng; chuyển sang worker phân tán theo trip, quản lý hàng đợi và dùng phần cứng decode nếu đo đạc cho thấy cần. **DuckDB/dbt trên một máy**, vì quét/join và ghi dữ liệu nhiều xe cạnh tranh RAM/I/O; dùng engine phân tán như Spark cho biến đổi lớn trên lakehouse, vẫn giữ test chất lượng và partition hợp lý. **Airflow LocalExecutor và storage cục bộ**, vì một container giới hạn song song, tài nguyên và nhiều object ảnh nhỏ; tách scheduler/webserver, dùng executor có worker mở rộng, object storage bền vững và gom file metadata/ảnh khi phù hợp. Airflow điều phối các job, không nên vận chuyển toàn bộ payload video qua scheduler. Quy mô 500 xe cần đo dung lượng/ngày và throughput trước khi chọn số worker.

## Câu hỏi riêng của bạn trên Metabase

**Trip nào có tỷ lệ khung bị loại cao nhất?** SQL đã lưu trong `report/metabase_question.sql`, thêm vào dashboard ở `http://localhost:3000/dashboard/2`, card 46:

```sql
SELECT drive_id, trip_id, frames, frames_kept,
       frames - frames_kept AS frames_dropped,
       ROUND(100.0 * (frames - frames_kept) / NULLIF(frames, 0), 2) AS dropped_pct,
       avg_speed_kmh, split
FROM marts.mart_trip_summary
ORDER BY dropped_pct DESC, trip_id;
```

Kết quả thực tế 26 trip: cao nhất là **drive_a / trip-04622**, 60 khung, giữ 26, loại **34 (56,67%)**, tốc độ trung bình 67,1 km/h. Tiếp theo `trip-03904`: **11/22 (50%)**; `trip-02086`: **28/60 (46,67%)**. Câu hỏi giúp ưu tiên kiểm tra trip mất nhiều dữ liệu; truy vấn này không tách nguyên nhân loại, nên không thể quy mọi khung bị loại cho camera đứng hình chỉ từ tỷ lệ. Bằng chứng: `metabase_question.json` và `metabase_dashboard.png`.

## Nhìn lại

Quan sát đáng chú ý: thí nghiệm chia theo frame làm cả 26 trip có mặt ở train và test, dù tỷ lệ split trông hợp lý. Ngưỡng dHash 6 loại nhiều khung hơn đáng kể so với 2, nhưng riêng số lượng không chứng minh chất lượng tốt hơn. Lỗi CRLF trên Windows làm script khởi động không chạy và làm checker checksum báo khác bản gốc; chuyển LF đã khôi phục checksum đúng. Cần kiểm tra thêm khả năng cùng tuyến đường xuất hiện ở các trip khác nhau, hoặc GPS có giá trị hợp kiểu nhưng sai vị trí thực; các test hiện tại chưa chứng minh đã bắt hai lỗi này. Nội dung được tổng hợp với hỗ trợ Codex và cần được sinh viên đọc, đối chiếu trước vấn đáp.
