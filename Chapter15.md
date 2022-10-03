# Chương 15: Thiết kế google drive

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

- Tính năng quan trọng: Upload, download files, file sync, notifications
- Hỗ trợ cả mobile app và web app
- Hỗ trợ mọi loại file format
- File trong storage cần được mã hoá
- File size <= 10GB
- DAU: 10 triệu
- Có thể share files
- Send notification khi file được chỉnh sửa, deleted hoặc shared
- Reliability: đây là non-functional requirement rất quan trọng, việc để mất files là không được phép
- Fast sync speed: việc file sync tốn nhiều thời gian sẽ làm ảnh hưởng đến trải nghiệm của người dùng
- Bandwidth usage: nếu hệ thống sử dụng nhiều băng thông không cần thiết cũng sẽ gây ảnh hưởng xấu đến người dùng (đặc biệt là trên mobile)
- Scalability: hệ thống nên có khả năng mở rộng để có thể xử lí lượng traffic lớn
- High availability

### Envelope estimation

- Giả sử hệ thống có 50 triệu người dùng và DAU là 10 triệu
- Users có 10GB free space
- Giả sử trung bình 1 user upload 2 files / ngày (kích cỡ trung bình của file là 500KB)
- 1:1 là tỉ lệ read/write
- Tổng số bộ nhớ cần cấp: `50 triệu * 10GB = 500 PB`
- QPS cho upload API: `10 triệu * 2 upload / 24 giờ / 3600s = 240`
- Peak QPS = QPS * 2 = 480

## Bước 2: High level design

Chúng ta sẽ bắt đầu bước này với việc thiết kế mọi thứ trên một server.

- Một server có khả năng upload và download files
- DB để theo dõi metadata như: `login info`, `file info`, ...
- File storage system (với dung lượng ban đầu là 1TB)

Ta sử dụng folder root là `drive/` cho File storage. Phía dưới là các `namespaces` dùng cho từng user.

Files được lưu trên server sẽ có cùng tên như files gốc

### APIs

Về cơ bản ta cần 3 APIs:

- Upload file
- Download file
- Get file revision

#### 1. Upload file lên Google drive

Có 2 loại upload:

- Simple upload: sử dụng khi file size nhỏ
- Resumable upload: sử dụng khi file size lớn và khi khả năng mạng gặp sự cố cao

VD về resumable upload API: `https://api.com/upload?type=resumable`

Quá trình upload file sẽ bao gồm 3 bước như sau:

- Gửi initial request để lấy về `resumable URL`
- Upload data và monitor upload state
- Nếu upload gặp sự cố, resume upload

#### 2. Download file từ google drive

VD: `https://api.com/files/download`

params: file path

#### 3. Get file revisions

VD: `https://api.com/files/revisions`

params:

- path: file path
- limit: số lượng tối đa revisions sẽ được trả về

Mọi API đều yêu cầu authenticate user và sử dụng HTTPs, SSL sẽ bảo vệ dữ liệu giữa client và server

### Di chú từ 1 server

Khi storage system đầy thì cách giải quyết đầu tiên được nghĩ đến đó là sharding data (tức là lưu trên nhiều storage servers). Hình dưới đây là cách tiếp cận lựa chọn shard để lưu dựa theo `user_id`

![Screen Shot 2022-10-02 at 18 36 42](https://user-images.githubusercontent.com/15076665/193447788-8b48a7a2-a7c4-452b-9b50-af72e7c8a317.png)

Vấn đề về dung lượng bộ nhớ đã được giải quyết xong có một vấn đề khác đó là availability tức là khả năng hệ thống vẫn có thể hoạt động như ý muốn kể cả khi có một storage server bị "sập". Sử dụng Amazon S3 có thể là một giải pháp vì S3 cho phép:

- Replication trong cùng một region cũng như giữa các regions khác nhau

Điều này giúp ta có thể tránh được vấn đề "mất dữ liệu"

Để tránh vấn đề xảy ra trong tương lai (khi một storage server bị sập), bạn có thể sử dụng `load balancer` để phân tải đều lên các servers khác nhau và khi có một server bị sập thì request đến server đó sẽ được điều chuyển đi sang server khác

Khi load balancer được thêm vào thì việc thêm/ bớt web servers sẽ trở nên dễ dàng hơn tuỳ theo traffic load

`Metadata DB` sẽ được di rời ra khỏi server để tránh SPOF. Khi đó ta cũng cần setup replication và sharding cho DB

`File storage` - khi sử dụng S3 để đảm bảo `availability` và `durability` files sẽ được replicated sang 2 regions khác nhau.

Hệ thống lúc này trông sẽ như sau:

![Screen Shot 2022-10-02 at 18 46 30](https://user-images.githubusercontent.com/15076665/193448097-d862dd07-43e8-40bf-9e4d-47f29f03275c.png)

### Sync conflict

Conflict xảy ra khi nhiều users cùng chỉnh sửa 1 file xảy ra khá thường xuyên với Google drive. Hình dưới đây minh hoạ quá trình xảy ra conflict

![Screen Shot 2022-10-02 at 19 01 21](https://user-images.githubusercontent.com/15076665/193448582-eadc93cf-1eef-4250-b054-4c04520ee168.png)

Việc chỉnh sửa của user 1 được hoàn thành nhưng version sau chỉnh sửa của user 2 bị conflict. Lúc này có 2 cách xử lí chính:

- Merge user 2 version và version mới nhất hiện tại lại
- Override version mới nhất hiện tại bởi user 2 version

### High level design

![Screen Shot 2022-10-02 at 19 03 32](https://user-images.githubusercontent.com/15076665/193448672-c66bb5db-2d7c-44d2-98e8-7f19c01ff9bd.png)

**Block server**: upload các `block storage` lên `cloud storage`. `Block storage` ở đây chính là một đơn vị độc lập của file (file khi được upload sẽ được chia nhỏ thành các blocks - kĩ thuật này thường được sử dụng trong `cloud base environment`). Mỗi block sẽ có một hash value riêng và được lưu trong metadata DB.

Mỗi block sẽ được lưu trong S3 và để "tái cấu trúc" lại file, các blocks sẽ được ghép lại với nhau.

Về block size thì Dropbox quy định max là 4MB

**Cold storage** dùng để lưu các inactive data, ở đây sẽ là các files không được truy cập trong một khoảng thời gian dài

> Chú ý ở đây đó là `File` được lưu ở `cloud` còn `Metadata DB` chỉ lưu metadata mà thôi

**Offline backup queue** lưu các thông tin về file changes khi user offline. Khi user online trở lại thì file changes sẽ được sync tới cho client

## Bước 3: Deep dive design

### Block servers

Với các files lớn, thường xuyên được thay đổi thì việc upload toàn bộ file mỗi khi cập nhật sẽ gây lãng phí bandwidth, do đó ta có 2 cách tiếp cận sau để tiết kiệm network traffic nhất có thể:

- Delta sync: chỉ những block được update mới được sync thay vì toàn bộ file (có thể sử dụng sync algorithm để xử lí)
- Compression: nén file cũng sẽ giúp giảm đi network traffic. Tuy nhiên với từng loại file ta sẽ có cách nén khác nhau, ví dụ: `gZip` và `bzip2` chỉ dùng để nén file text, còn file images và video sẽ sử dụng cách nén khác

Hình dưới đây sẽ mô tả công việc của block server (split file, compress file blocks, encrypt file blocks, upload lên storage)

![Screen Shot 2022-10-02 at 19 17 22](https://user-images.githubusercontent.com/15076665/193449166-1c08a45c-4d8f-4097-94c8-fe1af37a2b32.png)

Hình sau mô tả delta sync với chỉ "block 2" và "block 5" được thay đổi thì chỉ có 2 blocks này được upload lên cloud storage mà thôi

![Screen Shot 2022-10-02 at 19 18 37](https://user-images.githubusercontent.com/15076665/193449221-b14e7ecc-dae2-4d94-8aa4-db33b449a35b.png)

### High consistency requirement

Hệ thống cần đảm bảo tính thống nhất cao, sẽ không bao giờ có chuyện các clients khác nhau sẽ xem được cácc file với nội dung khác nhau ở cùng một thời điểm.

Để đảm bảo tính thống nhất cao, hệ thống cần xem xét đến việc đảm bảo tính thống nhất cho:

- Metadata cache
- Database layer

Với `Memory cache` thì các replicas khác nhau có thể có dữ liệu khác nhau. Để đảm bảo tính thống nhất thì chúng ta cần chắc chắn những điều sau:

- Dữ liệu trong cache replicas và trong master phải đồng nhất
- Không cho phép cache ghi lên DB để đảm bảo cache và DB có dữ liệu đồng nhất

Do RDB có ACID (Atomicity, Consistency, Isolation, Durability) nên việc đảm bảo tính đồng nhất cao là không quá khó khăn như NoSQL (không hỗ trợ ACID nên cần phải tạo ACID bằng tay)

### Metadata Database

Hình dưới đây chỉ là minh hoạ đơn giản (bao gồm những bảng và những cột quan trọng nhất)

![Screen Shot 2022-10-03 at 8 22 10](https://user-images.githubusercontent.com/15076665/193480881-e3313de1-76fb-47fc-bc76-6dfada9a0a1c.png)

`Namespace`: root directory của user
`File`: lưu các thông tin liên quan đến file
`File_version`: các thông tin về version history của file. Các rows đều là read-only để đảm bảo tính trung thực của revison history
`Block`: lưu các thông tin liên quan đến file block. Một file (ở bất kì version nào) cũng đều có thể được tái tạo lại bằng cách sắp xếp các blocks theo một thứ tự nhất định

### Upload flow

Sẽ diễn ra như hình bên dưới

![Screen Shot 2022-10-03 at 8 28 35](https://user-images.githubusercontent.com/15076665/193481081-5090c502-9a1b-4d75-88cd-890b0fe6537d.png)

2 requests sẽ được gửi song song. 1 request sẽ add file metadata, 1 request sẽ upload file lên cloud storage

#### Add file metadata

Notify service sẽ thông báo cho các clients liên quan (client 2) về file đang được upload

#### Upload file to cloud storage

File sẽ được upload lên `block server`, block server sẽ cắt nhỏ file thành các blocks, nén và mã hoá block, upload chúng lên cloud storage.

Sau khi file được upload thì `cloud storage` sẽ trigger `callback call` đến API server để cập nhật `upload status` thành `uploaded` (trong metadata DB)

Flow khi chỉnh sửa file cũng tương tự như trên

### Download flow

Download file sẽ được trigger khi file có sự chỉnh sửa hoặc có file được thêm mới. Có 2 cách sau để client biết rằng có file được chỉnh sửa / thêm mới để từ đó tiến hành download file:

- Nếu client A online thì notification service sẽ push notification đến client A để thông báo rằng có sự thay đổi/ thêm mới file nên client A cần pull version mới nhất về
- Nếu client A offline thì dữ liệu sẽ được đưa vào cache, khi client online trở lại nó sẽ pull version mới nhất về

Khi nhận được thông báo có sự thay đổi, client sẽ call API server để lấy về `Metadata` sau đó gửi request lên block server để lấy về các blocks cần thiết cho quá trình "tái cấu trúc" file với version mới nhất

Quá trình này được minh hoạ như hình bên dưới:

![Screen Shot 2022-10-03 at 23 28 04](https://user-images.githubusercontent.com/15076665/193602789-a46bdaed-40e5-4052-ad0f-0ce4ece7eb10.png)


### Notification service

Để đảm bảo tính đồng nhất của dữ liệu, mọi sự thay đổi dưới local của file sẽ được thông báo cho các clients khác để giảm thiểu đi conflict.

Notification service được tạo ra nhằm thực hiện mục tiêu trên.

Ở high-level notification service có thể chuyển dữ liệu tới client dưới hình thức `event`. Dưới đây là một vài cách thức:

- Long polling (Dropbox sử dụng phương thức này)
- Websocket

2 phương thức trên đều hoạt động ổn, nhưng lựa chọn `long polling` sẽ hợp lí hơn với 2 lí do sau đây:

- Tương tác giữa `client` và `Notification service` không phải là `bi-directional` cụ thể là client sẽ `pull` notification data về chứ không có chiều ngược lại
- Websocket sẽ phù hợp hơn cho những chat app (cần bidirectional communication). Google drive cũng không gửi quá nhiều notify với một lượng lớn dữ liệu bên trong

Với long polling, client sẽ "chủ động" tạo `long polling connection` tới notification service. Khi phát hiện có file bị thay đổi thì connect sẽ **BỊ ĐÓNG LẠI**. **BỊ ĐÓNG LẠI** ở đây là để tạo connect tới `API server` nhằm mục đích `pull` phiên bản mới nhất của file về.

Sau khi nhận được response là phiên bản mới nhất của file hoặc `connect bị timeout` thì client sẽ ngay lập tức "mở" một `long polling connect` tới notification service

### Save storage space

Để đăng "tính tin cậy" thì một file sẽ được lưu trên nhiều data centers. Ta thấy rằng bộ nhớ ở các data centers sẽ "đầy" rất nhanh khi phải lưu **quá nhiều versions** của **quá nhiều files**. Sau đây là các phương pháp giúp tiết kiệm bộ nhớ lưu trữ (tiết kiệm cả chi phí):

- `De-couple blocks` - ta sẽ tiến hành loại bỏ đi các blocks không cần thiết ở account level nhằm tiết kiệm bộ nhớ. Hai blocks được coi là giống nhau nếu chúng có `hash value` bằng nhau
- Thực hiện các "chiến lược" backup thông minh:
  - 