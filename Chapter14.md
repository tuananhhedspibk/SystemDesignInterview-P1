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

Quá trình upload video sẽ bao hàm 2 flow con song song sau:

- Upload video
- Update video metadata (video format, user infor, video size, ...)

Song song với việc upload video, client cũng request để update metadata, request bao gồm các thông tin như (video format, size, ...) API server sẽ cập nhật DB và cache

![Screen Shot 2022-10-01 at 21 46 32](https://user-images.githubusercontent.com/15076665/193410342-5406a976-4ae0-44d0-a0bf-03ec6287ecf3.png)

### Video streaming flow

Streaming khác với download ở chỗ download tức là ta copy nguyên si video từ source về user device còn streaming là quá trình lấy dữ liệu từ source video về user device liên tục.

Khi ta xem video trên youtube, youtube web sẽ load video từng chút từng chút về một cách liên tục nên ta có thể xem video ngay lập tức và không bị ngắt quãng

Trước khi đi vào streaming flow ta cần nắm một khái niệm quan trọng đó là `streaming protocol` - là cách chúng ta sẽ điều khiển luồng dữ liệu luân chuyển khi tiến hành streaming. Một streaming protocol phổ biến là `MPEG-DASH`

Các streaming protocol khác nhau sẽ hỗ trợ `video encoding`, `playback player` khác nhau. Khi tiến hành thiết kế một video streaming service ta cần chọn `streaming protocol` phù hợp cho usecase của chúng ta

Video sẽ được stream từ CDN, edge server gần nhất sẽ truyền video đến cho client giúp giảm tối đa thời gian trễ

## Bước 3: Deep dive design

### Video transcoding

Khi bạn thu một video trên điện thoại của bạn, video sẽ được xuất ra ở một format nhất định, muốn video này cũng được "mượt" trên thiết bị khác thì video phải được encode với format và bitrate phù hợp (bitrate là tỉ lệ bit được xử lí / thời gian - về lí thuyết thì bitrate càng cao thì video sẽ càng mượt)

High bitrate stream cần internet tốc độ cao và các xử lí phức tạp

Video transcoding quan trọng vì những lí do sau:

- Raw video với chất lượng cao có thể yêu cầu đến 60 frames / giây, nên để lưu trữ video có thể tốn tới cả trăm GB bộ nhớ
- Rất nhiều devices chỉ hỗ trợ một số định dạng video nhất định. Do đó việc transcoding sẽ giúp đảm bảo về format trên các devices khác nhau
- Để đảm bảo user có thể xem video chất lượng cao và có trải nghiệm xem video "mượt" nhất có thể thì ta sẽ stream những `video có độ phân giải cao` cho những user có `băng thông mạng rộng` còn `video có độ phân giải thấp` cho users `có băng thông thấp`
- Network condition có thể bị thay đổi thường xuyên (đặc biệt là trên mobile), do đó ta có thể chỉnh sửa chất lượng độ phân giải video (tự động hoặc bằng tay) tuỳ theo network condition của device

Các format types khác nhau đều có cấu trúc gồm 2 phần:

- Container: chứa video file, audio, metadata (format của container chính là file extension, ví dụ như: .mp4, .mov)
- Codecs: đây là giải thuật `nén` & `giải nén` với mục đích làm giảm kích cỡ video khi tiến hành điều chỉnh chất lượng video

### Directed acyclic graph (DAG) model

Transcoding là quá trình tốn kém chi phí và thời gian. Bên cạnh đó thì content creator có thể có những yêu cầu như:

- Watermark cho video
- Một số tự mình upload thumbnails
- Một số upload video chất lượng cao còn một số thì không

