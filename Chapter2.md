# Back-Of-The Envelope Estimation

Khi tiến hành phỏng vấn system design, ta sẽ được hỏi những câu đại loại như:

- Estimate system capacity hoặc performance requirements sử dụng back-of-the envelope estimation.

Những concepts sau ta cần nắm vững:

- Power of two
- Latency numbers
- Availability numbers

## Power of two

Data volume unit luôn là số mũ của 2. Một kí tự ASCII sử dụng 1 byte bộ nhớ (8bits nhớ)

## Latency numbers

- Memory có thể nhanh, nhưng disk sẽ chậm
- Tránh việc tìm kiếm trên disk nhiều nhất có thể
- Các giải thuật nén dữ liệu đơn giản thường khá nhanh
- Trước khi gửi dữ liệu nên nén dữ liệu nếu có thể
- Các data centers thường nằm ở các regions khác nhau, việc truyền dữ liệu giữa chúng cũng tốn thời gian

## Availability numbers

High availability hay còn gọi là tính sẵn sàng của hệ thống - thể hiện ở khả năng hệ thống có thể hoạt động liên tục trong một khoảng thời gian dài bao nhiêu.

High availability thường được đo bằng %, nếu bằng 100% tức có nghĩa là hệ thống không bị sập.

> Trong thực tế thì high availability sẽ nằm trong khoảng 99% - 100%

Service level agreement (SLA): là con số cam kết giữa service provider với customer về mức độ hoạt động của service. VD: với Cloud service của Amazon sẽ là 99.9% hoặc hơn thế. SLA thường được định nghĩa bởi các số 9, càng nhiều số 9 thì hệ thống càng tốt

VD: 99,99% < 99,9999% < 99,9999999%

## Ví dụ về estimate twitter QPS (query per second) and storage requirements

Giả sử:

- 300 triệu Monthly Active Users (MAU)
- 50% trong số đó sử dụng Twitter hàng ngày
- Users post 2 tweets mỗi ngày (trung bình)
- 10% tweets bao hàm nội dung media
=> Data được lưu trữ trong vòng 5 năm = ??? (media content only)

Estimation:

- DAU = 300 triệu * 50% = 150 triệu
- Tweets QPS = 150 triệu *2 / (24h* 3600s) ~ 3500
- Peak QPS = 2 *QPS ~ 3500* 2 ~ 7000

Size trung bình của 1 tweet

- tweet_id: 64 bytes
- text: 140 bytes
- media: 1mB

=> Media storage: 150 triệu *2* 10% *1MB = 30TB / ngày
=> 5 năm: 5* 365 * 30 ~ 55PB

## Tips

- Back-of-the-envelope estimation chủ yếu nói về quá trình. Xử lí vấn đề nhiều khi quan trọng hơn việc đưa ra kết quả.
- Interviewers có thể sẽ kiểm tra kĩ năng xử lí vấn đề quả bạn

Dưới đây là một vài tips cần lưu ý:

- Làm tròn và xấp xỉ: trong quá trình phỏng vấn sẽ rất khó để tính chính xác kết quả "99987 / 9.1", vậy nên hãy làm tròn và tính xấp xỉ để đưa ra con số trung bình (~ 100000 / 10).
- Viết ra những giả thiết của bạn để sau này có thể tham khảo lại
- Ghi rõ đơn vị, việc viết "5" sẽ gây khó hiểu do không rõ là "5MB" hay "5KB"
- Các back-of-the-envelope estimations hay bị hỏi: QPS, peak QPS, storage, cache, number of servers, ...
