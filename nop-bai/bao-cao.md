# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

<!--
HƯỚNG DẪN - đọc rồi XÓA TOÀN BỘ các khối chú thích này sau khi điền xong:

  - Giới hạn: KHÔNG QUÁ 1 TRANG A4, tương đương khoảng 450 - 550 từ nội dung.
  - Chỉ điền vào các chỗ ___ và các ô trong bảng. Không thêm mục mới.
  - Viết bằng câu hoàn chỉnh, không gạch đầu dòng cụt lủn.
  - Kiểm tra độ dài sau khi đã xóa hết chú thích:
        wc -w nop-bai/bao-cao.md
    và xem trước bản in bằng cách mở file trên GitHub rồi Ctrl+P / Cmd+P.
-->

| | |
|---|---|
| Họ và tên | Trịnh Bá Khánh Trinh |
| MSSV | 2A202601531 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/khanhtrinh2/K4_Track2_Day21_CI_CD_for_AI_Systems_TrinhBaKhanhTrinh_2A202601531 |
| Ngày nộp | 21/08/2026 |
---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

<!-- Khoảng 120 - 150 từ. Điền kết quả thật từ MLflow UI ở Bước 1, tối thiểu 3 lần chạy. -->

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |
| 4 | 200 | 0.2 | 3 | 0.7032 | 0.8700 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Lần 3 đạt f1_score cao nhất (0.7149), vượt ngưỡng 0.65 an toàn. Lần có accuracy cao nhất là lần 1 (0.8780), không trùng lần có f1_score cao nhất — cho thấy accuracy không phải chỉ báo đáng tin. Rõ nhất ở lần 2: giảm cả n_estimators và learning_rate khiến f1_score rơi xuống 0.6051 (dưới ngưỡng) trong khi accuracy chỉ giảm nhẹ — accuracy che giấu sự sụt giảm trên lớp thiểu số. Đánh đổi quan sát được: learning_rate thấp cần nhiều cây hơn để bù lại; lần 2 cả hai cùng giảm nên underfit, lần 3 tăng max_depth và n_estimators giúp mô hình học tốt hơn.

<!--
Trả lời trong phần Lý do:
  - Vì sao bộ này tốt hơn các bộ còn lại (dựa trên f1_score, không phải accuracy)?
  - Lần chạy có accuracy cao nhất có trùng với lần có f1_score cao nhất không?
    Nếu không, điều đó nói lên điều gì?
  - Bạn quan sát thấy đánh đổi nào giữa n_estimators và learning_rate?
-->

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

<!-- Khoảng 120 - 150 từ. -->

Tập dữ liệu Adult mất cân bằng: chỉ 24,8% mẫu thuộc lớp thu nhập cao. Một mô hình vô dụng luôn trả lời "thu nhập thấp" vẫn đạt accuracy 0,752 — cao đến mức gây hiểu nhầm, dù không học được gì và f1_score của lớp dương bằng 0. F1 của lớp dương đo trung bình điều hòa giữa precision và recall trên chính lớp thiểu số — lớp dễ bị bỏ qua nhất. Vì vậy f1_score phản ánh đúng khả năng phát hiện nhóm thu nhập cao, còn accuracy bị lớp đa số chi phối. Không dùng `average="weighted"`/`"macro"` vì hai cách này lấy trung bình theo cả hai lớp, bị lớp đa số kéo điểm lên cao, làm mất ý nghĩa giám sát riêng cho lớp dương.

<!--
Cần nêu được:
  - Phân bố lớp của tập dữ liệu (tỷ lệ lớp thu nhập > 50K) và hệ quả của nó.
  - Accuracy của một mô hình luôn trả lời "thu nhập thấp" là bao nhiêu, vì sao con số
    đó gây hiểu nhầm.
  - F1 của lớp dương đo điều gì mà accuracy không đo được.
  - Vì sao KHÔNG dùng average="weighted" hay average="macro" khi gọi f1_score.
-->

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

<!-- Nêu 2 - 3 khó khăn thật, mỗi ô một câu ngắn. -->

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| `ModuleNotFoundError: pkg_resources` khi chạy `train.py` | `setuptools` bản mới trong `.venv` đã loại bỏ `pkg_resources` mà `mlflow` vẫn cần | Downgrade `setuptools<81` |
| MLflow UI báo "No runs logged" dù train đã chạy xong | Train chạy ở Git Bash, `mlflow ui` mở ở PowerShell khác — biến môi trường không chia sẻ giữa hai shell | Chạy toàn bộ lệnh trong cùng một terminal, set lại biến rồi chạy lại |
| Lỗi `artifact scheme 'sqlite' is invalid` khi log model | Experiment `Default` được tạo với `artifact_location` mặc định dùng scheme proxy, không hợp với việc gọi script trực tiếp (không qua server) | Sửa `artifact_location` trong `mlflow.db` trỏ về thư mục cục bộ |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

<!-- Lấy số liệu từ bảng ở mục 3.6 của tasks/buoc-3.md. -->

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | ___ | ___ |
| Bước 3 (thêm `train_batch2`) | ___ | ___ |

**Nhận xét:** ___

<!--
Một câu trả lời trung thực kiểu "f1 giảm 0,01 vì dữ liệu mới cùng phân phối, không mang
thêm thông tin mới" được đánh giá cao hơn kết luận sai rằng thêm dữ liệu luôn tốt hơn.
-->

---

## 5. Phần Bonus Đã Thực Hiện (nếu có)

<!-- Xóa cả mục 5 nếu không làm bonus. Mỗi bonus tối đa 1 dòng. -->

- [ ] Bonus 1 - Tracking MLflow từ xa với DagsHub: ___
- [ ] Bonus 2 - Điều chỉnh ngưỡng quyết định: ___
- [ ] Bonus 3 - Báo cáo precision / recall tự động: ___
- [ ] Bonus 4 - Hoàn trả về phiên bản trước: ___
- [ ] Bonus 5 - Cảnh báo lệch lạc dữ liệu: ___
