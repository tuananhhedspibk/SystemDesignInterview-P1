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

Phần estimate dưới đây dựa theo "giả thiết cá nhân" khá nhiều, do đó các số liệu cụ thể vẫn phải xác nhận với interviewer

- DAU: 5 triệu người
- 1 user xem 5 videos / ngày
- 10% user upload video
- Kích cỡ trung bình của video là 300MB
- Trong ngày cần: `5,000,000 * 10% * 300MB = 150TB` để lưu trữ video mới
- CDN cost:
  - Chi phí CDN cho mỗi video được tính dựa theo lượng data được truyền ra khỏi CDN
  - Chúng ta lấy ví dụ với Amazon Cloudfront, tại Mỹ giá cho mỗi GB là 0.02$, vậy ta cần: `5 triệu * 5 * 0.3GB * 0.02$ = 150,000$` cho mỗi ngày

Ta thấy chi phí CDN cho videos là khá lớn dù các nhà cung cấp dịch vụ CDN có giá ưu đãi cho các big customer.

## Bước 2: High-level design

Như đã nói ở phần trước, ta sẽ tận dụng tối đa các Cloud service có sẵn để giảm đi độ phức tạp lúc triển khai. Ở đây chúng ta sẽ sử dụng CDN và blob storage của AWS. Câu hỏi đặt ra là tại sao không tự phát triển, thì lí do là:

- System design interview sẽ **KHÔNG ĐI CHI TIẾT VÀO CÁCH HOẠT ĐỘNG CỦA CÔNG NGHỆ** mà thay vào đó sẽ là **VIỆC CHỌN ĐÚNG CÔNG NGHỆ ĐỂ ĐẠT ĐƯỢC MỤC TIÊU ĐỀ RA** mà thôi. Hơn nữa trong khoảng thời gian phỏng vấn có hạn thì việc mô tả chi tiết về cách hoạt động của CDN và blob storage sẽ rất mất thời gian
- Việc tự thiết kế một CDN và blob storage có khả năng mở rộng cao cũng như tiết kiệm chi phí là rất phức tạp. Bản thân các công ty công nghệ nổi tiếng như Netflix cũng sử dụng AWS, hay Facebook sử dụng Akamai's CDN

Ở high-level, hệ thống sẽ chỉ bao gồm 3 components như sau:

![Screen Shot 2022-10-01 at 19 44 00](https://user-images.githubusercontent.com/15076665/193405823-5d20e6da-dd55-4f56-b83f-1aea3a2ecbc8.png)

everthing else ở trên bao hàm:

- Authentication
- Get video metadata
- ...

Interviewer có thể sẽ quan tâm đến 2 flows sau:

- Video uploading flow
- Video streaming flow

### Video uploading flow

High level uploading flow sẽ như dưới đây:

!["Screen Shot 2022-10-01 at 19 49 08](https://user-images.githubusercontent.com/15076665/193406037-231b300b-597c-47a6-9738-f0719587ecf0.png)

- Original storage: `blob storage system` được sử dụng để lưu trữ các videos gốc. Blob là "tập hợp binary data" được lưu trong DB dưới hình thức **một entity đơn lẻ**
- Transcoding server: **transcoding video** hay còn gọi là **encoding video**, đây là quá trình convert video format sang các dạng khác (MPEG) với mục đích tạo ra trải nghiệm stream video "mượt" nhất trên các loại devices cũng như băng thông khác nhau
- Transcoded storage: `blob storage` lưu transcoded videos
- CDN: các videos được cache trong CDN, khi ta nhấn play thì video sẽ được stream về từ CDN
- Completion queue: message queue lưu thông tin về event "transcoding video complete"
- Completion handler: các workers sẽ pull event từ `completion queue` về và update metadata trong DB và cache
