# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Trịnh Bá Khánh Trinh |
| MSSV | 2A202601531 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/khanhtrinh2/K4_Track2_Day21_CI_CD_for_AI_Systems_TrinhBaKhanhTrinh_2A202601531 |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |
| 4 | 200 | 0.2 | 3 | 0.7032 | 0.8700 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Lần 3 đạt f1_score cao nhất (0.7149), vượt ngưỡng 0.65 an toàn. Lần có accuracy cao nhất là lần 1 (0.8780), không trùng lần có f1_score cao nhất — cho thấy accuracy không phải chỉ báo đáng tin. Rõ nhất ở lần 2: giảm cả n_estimators và learning_rate khiến f1_score rơi xuống 0.6051 (dưới ngưỡng) trong khi accuracy chỉ giảm nhẹ — accuracy che giấu sự sụt giảm trên lớp thiểu số. Đánh đổi quan sát được: learning_rate thấp cần nhiều cây hơn để bù lại; lần 2 cả hai cùng giảm nên underfit, lần 3 tăng max_depth và n_estimators giúp mô hình học tốt hơn.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult mất cân bằng: chỉ 24,8% mẫu thuộc lớp thu nhập cao. Một mô hình vô dụng luôn trả lời "thu nhập thấp" vẫn đạt accuracy 0,752 — cao đến mức gây hiểu nhầm, dù không học được gì và f1_score của lớp dương bằng 0. F1 của lớp dương đo trung bình điều hòa giữa precision và recall trên chính lớp thiểu số — lớp dễ bị bỏ qua nhất. Vì vậy f1_score phản ánh đúng khả năng phát hiện nhóm thu nhập cao, còn accuracy bị lớp đa số chi phối. Không dùng `average="weighted"`/`"macro"` vì hai cách này lấy trung bình theo cả hai lớp, bị lớp đa số kéo điểm lên cao, làm mất ý nghĩa giám sát riêng cho lớp dương.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Service `income-api` trên VM crash liên tục, không pass health check khi Release job chạy | VM cài `scikit-learn` bản mới nhất (1.7.2) qua `pip3 install`, không khớp bản `1.4.2` dùng để train model, khiến `joblib.load` không đọc được file pickle | Cài đúng phiên bản `scikit-learn==1.4.2` trên VM để khớp với `requirements.txt` dùng khi train |
| MLflow UI báo "No runs logged" dù train đã chạy xong | Train chạy ở Git Bash, `mlflow ui` mở ở PowerShell khác — biến môi trường không chia sẻ giữa hai shell | Chạy toàn bộ lệnh trong cùng một terminal, set lại biến rồi chạy lại |
| Lỗi `artifact scheme 'sqlite' is invalid` khi log model | Experiment `Default` được tạo với `artifact_location` mặc định dùng scheme proxy, không hợp với việc gọi script trực tiếp (không qua server) | Sửa `artifact_location` trong `mlflow.db` trỏ về thư mục cục bộ |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`, 22.361 mẫu) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`, 44.722 mẫu) | 0.7354 | 0.8820 |

**Nhận xét:** f1_score tăng nhẹ (+0,0205) và accuracy cũng tăng (+0,008) khi gấp đôi dữ liệu huấn luyện. Khác với dự đoán rằng dữ liệu cùng phân phối sẽ không mang thêm thông tin, kết quả thực tế cho thấy 22.361 mẫu đầu chưa đủ để mô hình học bão hòa hết đặc trưng của lớp thiểu số (thu nhập cao) — gấp đôi kích thước mẫu giúp mô hình ước lượng ranh giới quyết định ổn định hơn, đặc biệt với các mẫu ở lớp thiểu số vốn ít đại diện hơn trong tập nhỏ. Mức tăng nhỏ (không đột biến) cũng phù hợp với kỳ vọng: hai nửa dữ liệu cùng nguồn nên phần lớn thông tin đã được học từ nửa đầu, chỉ còn cải thiện biên độ nhỏ chứ không tạo đột phá.

---

## 5. Phần Bonus Đã Thực Hiện

- [x] Bonus 1 - Tracking MLflow từ xa với DagsHub: kết nối repo GitHub với DagsHub, cấu hình `MLFLOW_TRACKING_URI`/`MLFLOW_TRACKING_USERNAME`/`MLFLOW_TRACKING_PASSWORD` (token) làm GitHub Secrets, job Train trong `cicd.yml` ghi log trực tiếp lên DagsHub MLflow server thay vì SQLite cục bộ.
