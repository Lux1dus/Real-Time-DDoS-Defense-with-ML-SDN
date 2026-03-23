# Tổng quan cấu trúc `source`

Tập hợp này được xây dựng cho dự án phòng chống DDoS kết hợp Machine Learning + SDN. Mỗi thư mục và file đóng vai trò riêng biệt trong quy trình thu thập, xử lý, huấn luyện và triển khai.

## 1. `ciclib/`
- Thư viện hỗ trợ xử lý dữ liệu CIC (CICIDS2017) và các hàm chuyên biệt cho bài toán DDoS.
- Có thể chứa các module tùy chỉnh lấy đặc trưng, tiền xử lý và tiện ích liên quan.

## 2. `controller/`
- Tập lệnh / logic chạy trên controller SDN (ví dụ Floodlight).
- `realtime_floodlight_ML.py`: kết nối thời gian thực với Floodlight, nhận dữ liệu mạng, gọi mô hình ML và ra quyết định chặn tấn công.

## 3. `dataset/`
- Chứa dữ liệu huấn luyện và kiểm thử đã được xử lý.
- `CICIDS2017_processed.csv`: dataset chính đã xử lý từ CICIDS2017, dùng cho huấn luyện và đánh giá mô hình.

## 4. `feature_importances.csv`
- Danh sách đặc trưng quan trọng sau khi huấn luyện (feature importances).
- Dùng để chọn đặc trưng, giải thích mô hình.

## 5. `machinelearning/`
- Mã nguồn xử lý ML, huấn luyện, lưu và nạp mô hình.
- `feature_schema.py`: định nghĩa tên cột, đặc trưng khi tiền xử lý.
- `ML_trainer.py`: tập lệnh huấn luyện mô hình từ dataset, tạo mô hình `model.pkl`.

## 6. `metadata.pkl`
- File nhị phân lưu metadata tiền xử lý/encoder (thường dùng để duy trì định dạng cột, scaler, encoder category).

## 7. `mininet/`
- Thành phần cấu hình/triển khai mạng ảo dùng Mininet.
- `topology.py`: định nghĩa topology SDN (switch, host, link).
- `ddos.sh`, `ping.csh`, `index.html`: script tấn công/kiểm tra/kết quả demo.

## 8. `model.pkl`
- Mô hình ML đã huấn luyện và serialize (pickle).
- Dùng trong `controller/realtime_floodlight_ML.py` để dự đoán lưu lượng DDoS.

## 9. `model_eval_cm.png`, `model_eval_pr.png`, `model_eval_roc.png`
- Hình trực quan hóa hiệu suất mô hình: confusion matrix, precision-recall, ROC.

## 10. `output/`
- Kết quả xử lý trong thời gian chạy thực:
  - `final_csv/Predict.csv`: dự đoán tấn công/normal.
  - `pcap_done/`, `pcap_in/`: các file pcap đầu vào/đã xử lý.

## 11. `processing/`
- Script xử lý file pcap, chuyển sang định dạng phân tích.
- `capture_tcpdump.sh`: chụp gói tin đầu vào.
- `pcap_processor.sh`: chuyển đổi pcap sang dữ liệu cho ML.

## 12. `venv/` và `.venv.zip`
- Môi trường ảo Python chứa thư viện phụ thuộc.
- `.venv.zip` lưu gói backup nhanh, dễ tái lập môi trường.

---

#### Hướng dẫn nhanh
- Bắt đầu từ `machinelearning/ML_trainer.py` để tái huấn luyện mô hình.
- Nạp `model.pkl` cùng `metadata.pkl` trong `controller/realtime_floodlight_ML.py` để chạy test SDN.
- Dùng `mininet/` cho mạng ảo và `processing/` với `dataset/` để tạo dữ liệu. 
