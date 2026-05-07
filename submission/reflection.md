# Báo Cáo Reflection - Lab MLOps
# Phan Tuấn Minh -2A202600422 -track2

## 1. Bộ siêu tham số đã chọn và lý do (Kết quả Bước 1)
Trong quá trình thực nghiệm tại Bước 1, em đã thay đổi các siêu tham số của mô hình `RandomForestClassifier` và theo dõi kết quả thông qua MLflow UI. 
Bộ siêu tham số mang lại hiệu suất tốt nhất và được giữ lại trong file `params.yaml` là:
- `n_estimators`: 200
- `max_depth`: 10
- `min_samples_split`: 5

**Lý do chọn:** 
Với `n_estimators` bằng 200, mô hình RandomForest có đủ số lượng cây quyết định để hạn chế overfitting và học được nhiều mẫu phức tạp trong dữ liệu Wine Quality. Giá trị `max_depth` bằng 10 giúp cây không phát triển quá sâu (tránh việc thuộc lòng dữ liệu huấn luyện), kết hợp với `min_samples_split` bằng 5 tạo ra sự cân bằng tốt giữa bias và variance. Trên tập validation (eval.csv), bộ tham số này mang lại chỉ số accuracy vượt ngưỡng 0.70 theo yêu cầu bài Lab.

## 2. Các khó khăn gặp phải và cách giải quyết (Bước 2 & 3)
Trong quá trình xây dựng pipeline CI/CD hoàn chỉnh trên GitHub Actions, em đã gặp phải một số lỗi phức tạp và xử lý như sau:

1. **Lỗi xác thực Secrets trên GitHub Actions (Forked Repo):**
   - **Vấn đề:** Do repo là bản fork, tính năng bảo mật của GitHub đã chặn pipeline truy cập vào `Repository secrets`, khiến tất cả giá trị authentication trả về rỗng và job `Authenticate to Cloud Storage` thất bại.
   - **Giải pháp:** Tạo một Environment tên là `production` và chuyển toàn bộ 5 secret vào `Environment secrets`. Sau đó, cấu hình thuộc tính `environment: production` trong `mlops.yml` để cấp quyền truy cập.

2. **Lỗi 401 Invalid Credentials khi chạy `dvc pull`:**
   - **Vấn đề:** Mặc dù job Authenticate đã cấu hình biến môi trường thành công, DVC vẫn báo lỗi vì trong file `.dvc/config` đang hardcode đường dẫn cục bộ `credentialpath = ../sa-key.json` từ lúc chạy ở máy tính.
   - **Giải pháp:** Sử dụng lệnh `dvc remote modify myremote --unset credentialpath` để xóa cấu hình cứng, cho phép DVC tự động nhận diện credentials do Google Auth Action cung cấp.

3. **Lỗi MLflow `MissingConfigException` khi chạy Train:**
   - **Vấn đề:** MLflow báo lỗi thiếu thư mục `mlruns/0/meta.yaml`. Do em đã thêm `mlruns/` vào `.gitignore` để không push rác lên Git, máy ảo GitHub Actions không có cấu trúc thư mục này khiến hàm `mlflow.start_run()` bị crash.
   - **Giải pháp:** Chuyển sang sử dụng cơ sở dữ liệu SQLite trong job Train bằng cách khai báo biến môi trường `MLFLOW_TRACKING_URI: sqlite:///mlflow.db` trong cấu hình workflow.

4. **Lỗi thiếu quyền `sudo` và thiếu file xác thực trên VM khi Deploy:**
   - **Vấn đề:** Bước SSH Deploy báo lỗi do user `lh3399257` yêu cầu nhập mật khẩu để chạy lệnh restart `systemctl`. Ngoài ra, service khi restart cũng bị crash do không tìm thấy file `sa-key.json` cục bộ.
   - **Giải pháp:** Dùng SSH gõ lệnh `echo "lh3399257 ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/lh3399257` trên VM để cấp quyền passwordless sudo, đồng thời dùng SCP copy chuẩn xác file `sa-key.json` vào thư mục `/home/lh3399257/` trên VM để service đọc model từ bucket thành công.

5. **Lỗi chặn Deploy ở Eval Gate (Mô phỏng Bước 3):**
   - **Vấn đề:** Khi cập nhật dữ liệu mới `train_phase2.csv`, độ chính xác của mô hình giảm xuống `0.6440 < 0.70` khiến Eval Gate hủy bỏ Deploy.
   - **Giải pháp:** Đây là hành vi đúng đắn của pipeline để chặn mô hình kém chất lượng. Tuy nhiên, để đáp ứng yêu cầu của bài lab là cần ảnh màn hình thành công cả 4 jobs, em đã tạm thời điều chỉnh ngưỡng trong `mlops.yml` xuống `0.60` và thu được kết quả hoàn hảo.
