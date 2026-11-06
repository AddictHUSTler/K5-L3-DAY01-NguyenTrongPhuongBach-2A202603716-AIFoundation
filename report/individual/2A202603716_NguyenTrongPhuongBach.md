# Báo cáo cá nhân — Day 01: Lọc tin nhắn rác

**SEED:** 1806. Số liệu lấy từ JSON và output notebook đã lưu.

## 1. Bằng chứng

| Tệp | Trạng thái |
| --- | --- |
| [day01_lab.ipynb](day01_lab.ipynb) | Có output, 15/15 kiểm tra đạt |
| [lab1_report.json](lab1_report.json) | Có 10 dự đoán và kết quả |
| [hand_calc.jpg](hand_calc.jpg) | Có ảnh tính tay |

## 2. Kết quả chính

| Mô hình | F1 test | F1 SUB 40 | p95 ms/tin | USD/1.000 tin |
| --- | ---: | ---: | ---: | ---: |
| BoW + LogReg | 0.910 | 0.889 | 0.140 | 0 |
| BoW + MLP | 0.936 | 0.974 | 7.480 | 0 |
| MiniLM + kNN | 0.918 | 0.976 | 52.702 | 0 |
| MiniLM + LogReg | 0.950 | 0.974 | 33.869 | 0 |
| MiniLM + MLP head | 0.950 | 0.974 | 38.872 | 0 |
| LLM nhỏ zero-shot | — | 0.923 | 1614.510 | 0.007623 |
| LLM lớn zero-shot | — | 0.974 | 1547.302 | 0.076233 |
| LLM nhỏ few-shot | — | 0.923 | 1151.436 | 0.016290 |

LLM chỉ chạy trên SUB 40 tin. Chi phí 0 của mô hình local là phí API, chưa gồm hạ tầng.

## 3. Dự đoán và thực tế

- **Bỏ ReLU:** đoán F1 gần LogReg (0,910), có thể thấp hơn MLP. Thực tế F1 là 0,936, bằng MLP có ReLU ở mức làm tròn. Hai lớp Linear vẫn tương đương một phép biến đổi tuyến tính, nhưng cách tối ưu khác nên kết quả không nhất thiết bằng bản LogReg tự code.
- **Few-shot:** đoán có thể cải thiện phân loại. F1 vẫn là 0,923, trong khi phí tăng từ 0,007623 lên 0,016290 USD/1.000 tin. Thêm ví dụ chưa giúp ở lần đo này.
- **Stress test:** đoán LLM có thể khó với prompt injection, nhưng cả prompt thường và phòng thủ đều đúng 50/50. Bộ này chưa tìm ra lỗi của LLM, không đủ để kết luận an toàn với mọi injection. Ô dự đoán ghi rõ được bổ sung sau khi xem kết quả, nên không tính là dự đoán trước lần chạy cũ.

## 4. Phân tích 50 case khó

BoW + LogReg tụt nhiều nhất: F1 SUB từ 0,889 xuống 0,286. Accuracy ở nhóm spam giọng tự nhiên là 0%, né từ khóa khoảng 17%. MiniLM + kNN đạt F1 0,759; MiniLM + LogReg đạt 0,640. Nhóm nói về spam khó với embedding, accuracy chỉ 25–50%.

- **Case #14:** tin kể chuyện làm tại nhà kiếm £300/ngày, không cần kinh nghiệm. Nhãn là spam nhưng BoW + LogReg đoán ham (cả nhóm này đều sai): giọng kể cá nhân thiếu từ báo hiệu spam quen thuộc.
- **Case #19:** tin thân mật “Hey sweetie, missed you lately…” rồi hứa gửi link. BoW + LogReg cũng bỏ sót: các từ thường gặp trong hội thoại che đi ý dụ người nhận phản hồi.

Hai bản LLM đều đạt F1 1,000; chưa thấy prompt phòng thủ cải thiện thêm. Bộ 50 case dùng để tìm điểm yếu, không đại diện tần suất spam thực tế.

## 5. Thử thách cá nhân

SEED 1806 nhận đề #1: đổi ngưỡng BoW + LogReg. Giữ nguyên mô hình, tính xác suất trên test rồi áp dụng từng ngưỡng.

| Ngưỡng | Precision | Recall | F1 |
| --- | ---: | ---: | ---: |
| 0,2 | 0,921 | 0,911 | 0,916 |
| 0,5 | 0,987 | 0,844 | 0,910 |
| 0,8 | 1,000 | 0,622 | 0,767 |

Ngân hàng sợ chặn nhầm OTP nên ưu tiên 0,8: lần đo này không có false positive, đổi lại bỏ sót nhiều spam hơn. Chưa thể bảo đảm mọi OTP thực tế đều được giữ lại.

## 6. Phương án cho sản phẩm thật

Chọn BoW + MLP làm bộ lọc chính: F1 test 0,936, p95 7,480 ms, throughput cả lô khoảng 8.811 tin/giây, so với nhu cầu trung bình khoảng 116 tin/giây. Đổi lại, F1 trên case khó chỉ 0,450 nên cần thêm dữ liệu và kiểm tra trước triển khai.

BoW + LogReg, BoW + MLP và hai đầu LogReg/MLP của MiniLM đều đạt p95 < 50 ms trong phép đo. MiniLM + LogReg có F1 test cao nhất (0,950), nhưng throughput khoảng 36 tin/giây trên cấu hình đo nên cần tăng năng lực xử lý. LLM vượt SLA; có thể dùng để rà soát bất đồng bộ các tin chưa chắc chắn.

Với 10 triệu tin/ngày, 30 ngày/tháng, ngoại suy phí token đã đo: khoảng 2.287 USD/tháng cho LLM nhỏ zero-shot, 22.870 USD cho LLM lớn và 4.887 USD cho few-shot. Chưa gồm retry và hạ tầng.

## 7. AI log và điều đã học

| Yêu cầu với AI | AI hỗ trợ | Kiểm chứng / phần còn cần làm |
| --- | --- | --- |
| Kiểm tra notebook mới, sửa theo output, giữ ngắn gọn (log đã có trong notebook). | Cập nhật đáp án F1, SLA, stress test, chi phí và ngưỡng. | Theo log cũ: đối chiếu output, kiểm tra cú pháp và phép tính; không gọi lại API. Một số nhận xét vẫn dùng số độ trễ cũ. |
| Điền báo cáo từ JSON và notebook, đổi tên notebook theo hướng dẫn nộp bài. | Tổng hợp báo cáo này và sắp xếp tệp nộp. | AI đối chiếu JSON với output đã lưu; người học cần đọc lại và đổi tên báo cáo theo MSSV/họ tên. |

F1 cao trên test thường chưa bảo đảm xử lý tốt spam mới: BoW + LogReg giảm từ 0,889 xuống 0,286 trên bộ khó. Tăng ngưỡng giảm chặn nhầm nhưng làm giảm recall. Few-shot và mô hình lớn hơn cần được đo trên cùng dữ liệu trước khi quyết định dùng.
