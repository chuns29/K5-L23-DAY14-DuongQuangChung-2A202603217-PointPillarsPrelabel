# Báo cáo Day 14 — Dương Quang Chung — 2A202603217


- Mã học viên dùng với `--student`: `217`
- `seed_tag` trong `frames_A.txt`: `393590736f`

## 1. Dự đoán trước sweep (bước 5b)

> Nội dung dự đoán dưới đây do AI hỗ trợ soạn từ ảnh và số đo tập A theo yêu cầu của người học; không trình bày là nhận xét tự viết độc lập. Được ghi trước khi chạy sweep và không sửa lại theo kết quả sweep.

- Tham số dự đoán giảm công sửa nhiều nhất: tăng `score_thr` từ `0.1` lên `0.4`; thử kết hợp `min_points=5` thay cho `0`.
- Lý do: cấu hình gốc trên ba frame cần xóa 72 box, thêm 5 và sửa 16. Đối chiếu camera cho thấy nhiều dự đoán thừa nằm trên cây, bụi hoa và hàng rào. Tăng ngưỡng tin cậy có thể loại bớt các dự đoán yếu; yêu cầu ít nhất 5 điểm trong hộp có thể loại thêm các hộp quá thưa điểm.
- Dự đoán: hai thay đổi trên có thể giảm chi phí tổng, nhưng chưa biết cấu hình nào tốt nhất. Thêm một box bị sót có trọng số `1.0`, còn xóa một box thừa chỉ có trọng số `0.3`, nên mất thêm một vật thật có thể xóa hết lợi ích của việc loại ba box thừa. Các vật xa, nhỏ hoặc bị che khuất có thể bị lọc nhầm. Vì vậy cần so sánh số xóa/thêm/sửa và leave-one-out sau sweep, không kết luận chỉ dựa vào số box giảm.
- Giới hạn của dự đoán: tập A chỉ có ba frame, nhãn tham chiếu có hỗ trợ AI/API và sáu box khó xác nhận đã bị loại theo quyết định của người học. Chi phí đo được phụ thuộc bộ tham chiếu này, không chứng minh cấu hình sẽ tốt trên mọi cảnh.

## 2. Tập A: sửa nhãn và tune

- Task CVAT: `D14-217-A` (task 28, job 28).
- Xuất nhãn bằng `scripts/vf_cvat.py reference`; chỉ có Shape, không có Track.
- `s01f44`: 17 box; `s03f41`: 7 box; `s04f47`: 10 box. Tổng 34 box.
- **Khai báo hỗ trợ AI/API:** Người học đã thao tác sửa một số box trên CVAT. Sau đó dùng Codex đối chiếu camera và LiDAR, xóa box thừa, chỉnh tâm/kích thước/đáy/hướng/lớp, đánh dấu che khuất và thêm box qua API CVAT. Sáu box LOW_VISIBILITY được xóa theo quyết định của người học. Vì vậy, không mô tả tập A là hoàn toàn sửa tay độc lập. Một số kích thước phần bị che được suy luận, không phải số đo chuẩn đã xác minh.
- Script hiển thị `scripts/cvat_lidar_display.js` chỉ hỗ trợ nhìn điểm, mặt đường tham chiếu và thao tác; các lệnh chỉnh nhãn API là bước riêng.

| Frame | Box tham chiếu | Xóa | Thêm | Sửa | Chi phí gốc |
|---|---:|---:|---:|---:|---:|
| s01f44 | 17 | 29 | 3 | 8 | 15.7 |
| s03f41 | 7 | 30 | 0 | 4 | 11.0 |
| s04f47 | 10 | 13 | 2 | 4 | 7.9 |


| Cấu hình | Xóa | Thêm | Sửa | Chi phí |
|---|---:|---:|---:|---:|
| Gốc (`score_thr=0.1`, `min_points=0`) | 72 | 5 | 16 | 34.6 |
| `best_config.json` (`score_thr=0.5`, `min_points=0`) | 10 | 11 | 12 | 20.0 |
| Dự đoán trước sweep (`score_thr=0.4`, `min_points=5`) | 27 | 9 | 13 | 23.6 |

- Tham số đã đổi trong `best_config.json`: `score_thr: 0.1 → 0.5`; các tham số khác giữ cấu hình gốc. `score_thr=0.5, min_points=5` đồng hạng 20.0; script lưu cấu hình `min_points=0` vì xuất hiện trước trong thứ tự thử.
- Kết quả leave-one-out từ script:

| Frame bỏ ra | Cấu hình chọn trên 2 frame còn lại | Chi phí trên frame bỏ ra | Oracle trong lưới |
|---|---|---:|---:|
| s01f44 | score_thr=0.5, min_points=0 | 11.4 | 11.4 |
| s03f41 | score_thr=0.5, min_points=0 | 3.8 | 3.8 |
| s04f47 | score_thr=0.5, min_points=0 | 4.8 | 4.8 |

- Số đo tự động: ba lượt đều chọn cùng cấu hình và đạt chi phí bằng oracle trong lưới đã thử. Điều này chỉ mô tả ba frame A, chưa đo khả năng tổng quát trên B.

- Cấu hình có ổn định trên từng frame không? Vì sao tôi vẫn chọn (hoặc không chọn) nó: ...

## 3. Trước/sau (bước 8)

![so sánh s01f44](../../d14_217/compare/s01f44.jpg)

- Số đo sweep: chi phí gốc 34.6; cấu hình chọn 20.0; mức giảm 42.2% trên tập A. Đây là chi phí theo công thức, không phải thời gian lao động. Cấu hình dự đoán 0.4/5 có chi phí 23.6, giảm 31.8%.
- Dự đoán ở mục 1 đúng hay sai? Giải thích bằng số box xóa/thêm/sửa: ...

## 4. Tập B

Số đo từ `scripts/pp_prelabel.py --frames d14_217/frames_B.txt --config d14_217/best_config.json --gt d14_217/ref_B.json`:

| Xóa | Thêm | Sửa | Chi phí |
|---:|---:|---:|---:|
| | | | |

- Loại box tốn công sửa nhất ở tập B: ...

## 5. Giới hạn

- Nêu ít nhất 2 giới hạn (ví dụ: chỉ 3 frame tune; model KITTI không có Truck/Bus; chi phí là công thức, chưa phải thời gian sửa thật): ...
