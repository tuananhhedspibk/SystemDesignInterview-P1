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

Cần có thêm abstraction level để:

- Tăng khả năng xử lí song song
- Hỗ trợ các pipeline processing khác nhau

Facebook hỗ trợ DAG model, ở đó họ định nghĩa các tasks trong stages để chúng có thể được thực thi "tuần tự" hoặc "song song".

> Nói một cách đơn giản thì "DAG model" chính là mô hình định nghĩa các tasks mà ở đó các tasks này có thể được thực thi "tuần tự" hoặc "song song"

Với hệ thống của chúng ta, chúng ta sẽ định nghĩa một DAG model như sau:

![Screen Shot 2022-10-02 at 11 56 25](https://user-images.githubusercontent.com/15076665/193435873-2fb3ec8a-3d66-4cf5-b771-0275ac4c1a6b.png)

Một video gốc luôn được chia thành:

- Video file
- Audio
- Metadata

Sau đây là một số tác vụ sẽ được áp dụng cho `video file`:

- `Inspection` - kiểm chứng chất lượng của video cũng như kiểm tra xem video có phải là `malformed` hay không
- `Video encoding` - video được convert để hỗ trợ các độ phân giải, codec, bitrate khác nhau
- `Thumbnail` - có thể được upload bởi user hoặc được gen bởi hệ thống
- `Watermark` - ảnh được đặt trên top layer của video để định danh các thông tin về video

Hình dưới đây mô tả một ví dụ về video encoding

![Screen Shot 2022-10-02 at 12 03 03](https://user-images.githubusercontent.com/15076665/193436028-ca0dc9ed-7634-4fe2-b0eb-ae77adce9b50.png)

### Video transcoding architecture

Kiến trúc dưới đây sẽ tận dụng các cloud services khác để triển khai

![Screen Shot 2022-10-02 at 12 05 46](https://user-images.githubusercontent.com/15076665/193436072-bc2a2b9f-ae8d-40af-9a6b-22a8b97af781.png)

#### Preprocessor

Có 4 nhiệm vụ sau đây:

①`Video splitting` - chia nhỏ video thành các GOP (Group of Picture). 1 GOP là 1 chuỗi các frames được sắp thứ tự. 1 GOP là một đơn vị độc lập có thể "xem được" (thường khoảng vài giây)
② Chia nhỏ video cho các devices đời cũ (do các devices này không hỗ trợ video splitting)
③ `DAG generation` - quá trình này dựa theo config file mà client programmer viết. Một DAG model đơn giản có thể như dưới đây:

![Screen Shot 2022-10-02 at 12 14 24](https://user-images.githubusercontent.com/15076665/193436351-6c24ece3-a842-4fb1-bfcf-bccb26d90bfb.png)

Dưới đây là 2 files config của 2 tasks trên

![Screen Shot 2022-10-02 at 12 20 45](https://user-images.githubusercontent.com/15076665/193436440-26140432-e5df-423e-b7f5-1fbccb5d0faa.png)

④ `Cache data` - Preprocessor sẽ lưu các segment videos vào `temporary storage` để trong trường hợp encoding fails thì hệ thống vẫn có thể lấy segment videos từ bộ nhớ tạm ra dùng cho quá trình retry (tăng reliability)

#### DAG scheduler

Chia nhỏ `DAG graph` thành các tasks ứng với các stages và đặt chúng vào `task queue` bên trong `Resource manager` như hình minh hoạ dưới đây

![Screen Shot 2022-10-02 at 12 25 49](https://user-images.githubusercontent.com/15076665/193436526-978f960b-24dc-46b5-922a-0afd17d85b3c.png)

#### Resource Manager

Bao gồm các `queues` và `task scheduler` đảm nhận việc phân bổ tài nguyên như hình bên dưới

![Screen Shot 2022-10-02 at 12 28 14](https://user-images.githubusercontent.com/15076665/193436581-0df898a2-1972-4094-8843-52fc4bc25dca.png)

- Task queue: priority queue bao gồm các tasks cần được thực thi
- Worker queue: priority queue bao gồm `utilization info` về các workers
- Running queue: lưu thông tin về task và worker đang thực thi task đó
- Task scheduler:
  - Lấy task sẽ được thực thi dựa theo mức độ ưu tiên (cao nhất) cũng như worker (tối ưu nhất) sẽ sử dụng để thực thi task đó
  - "Hướng dẫn" cho worker cách thực hiện task
  - Các thông tin về `task/worker` sẽ được bind và đưa vào running queue
  - Loại bỏ thông tin về job (worker thực thi task) khi job đã hoàn thành

#### Task workers

Chạy các tasks được định nghĩa bởi DAG. Các task workers khác nhau có thể chạy các tasks khác nhau

![Screen Shot 2022-10-02 at 12 37 48](https://user-images.githubusercontent.com/15076665/193436800-191134bf-e26e-499d-891d-00c1c458297d.png)

#### Temporary storage

Multiple storage systems được sử dụng ở đây. Việc lựa chọn `storage system` sẽ tuỳ thuộc vào các yếu tố như:

- Data type
- Data size
- Access frequency
- ...

Với metadata có 2 đặc điểm sau:

- Kích thước nhỏ
- Thường xuyên được truy xuất bởi worker

thì ta có thể caching nó trong memory. Còn với video và audio có kích cỡ lớn hơn ta sẽ sử dụng `blob storage system`. Dữ liệu được lưu trong `temporary storage` sẽ được giải phóng khi quá trình xử lí video tương ưng được hoàn tất

#### Encoded video

Output của flow (VD: test.mp4)

### System optimizations

#### Speed optimization: parallelize video uploading

Trong thực tế, ta sẽ không upload toàn bộ video lên một lúc mà sẽ chia nhỏ video thành các GOPs như hình dưới đây

![Screen Shot 2022-10-02 at 13 00 41](https://user-images.githubusercontent.com/15076665/193437256-be6bfba9-fcf1-4469-a539-ad5e4cf2b67e.png)

Việc chia nhỏ video thành các GOPs sẽ được triển khai bởi phía client để tăng tốc độ upload

![Screen Shot 2022-10-02 at 13 02 16](https://user-images.githubusercontent.com/15076665/193437319-16225b17-43ad-47d7-8526-c8c8f6f01ae5.png)

#### Speed optimization: place upload centers close to users

Triển khai các `upload centers` gần với người dùng. Ở đây ta có thể sử dụng CDN như `upload centers`

VD: Người dùng ở Mỹ sẽ upload video lên `North American center`, ở Ấn độ, Trung Quốc sẽ upload lên `Asia center`, ...

#### Speed optimization: parallelism everywhere

Để tăng tốc bằng cách xử lí song song ta có thể tiến hành tạo thêm một system mới song song với system hiện có.

Muốn triển khai xử lí song song thì thiết kế ban đầu cần phải được sửa đổi

Hình dưới đây minh hoạ quá trình "vận chuyển" video từ `original storage` đến `CDN` (chú ý rằng output sẽ phụ thuộc vào input của bước trước) của thiết kế hiện nay

![Screen Shot 2022-10-02 at 13 10 18](https://user-images.githubusercontent.com/15076665/193437470-b329cf2e-3b11-408a-8f01-35b104b298e6.png)
