# Chương 14: Thiết kế Youtube

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Vì Youtube là một nền tảng chia sẻ video "cực lớn", ngoài tính năng chính là chia sẻ video, youtube còn có các tính năng khác như: comment, like, share video, ... Do đó thiết kế toàn bộ các tính năng này trong vòng 45-60 phút là điều không thể. Vậy nên cần xác nhận với interviewer về yêu cầu hệ thống trước khi đi vào thiết kế

- Tính năng quan trọng: Upload và xem video
- Hỗ trợ: mobile app, smart tv, web
- DAU: 5 triệu
- Thời gian trung bình 1 user dành cho hệ thống / ngày: 30 phút
- Hỗ trợ users đến từ nhiều nơi trên thế giới
- Hỗ trợ mọi video format
- Yêu cầu mã hoá
- Hệ thống tập trung vào video cỡ nhỏ và vừa. Hỗ trợ max size là 1GB
- Nên sử dụng các dịch vụ cloud có sẵn như AWS, GCP, ... (build from scratch rất tốn thời gian và chi phí)
- Video streaming phải "mượt"
- Low infra cost
- High availability, scalability, reliability

### Back of the envelop estimation
