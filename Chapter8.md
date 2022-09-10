# Chương 8: Design URL Shortener

Trong chương này ta sẽ thiết kế một hệ thống generate các short URL tương tự như tinyurl

## Bước 1: Hiểu vấn đề và xác định phạm vi thiết kế

Một trong những yếu tố cơ bản nhất trong quá trình interview về system design đó chính là những câu hỏi làm rõ requirements.

Với URL shortener ta cần quan tâm tới:

- Format của Shorten URL, ví dụ với URL: <https://www.systemdesigninterview.com/q=chatsystem&c=loggedin&v=v3&l=long> hệ thống sẽ cho ra shorten URL: <https://tinyurl.com/y7keocwj>. Nếu click vào shorten URL, người dùng sẽ được redirect về URL gốc
- 100 triệu URLs cần được gen mỗi ngày
- URL cần ngắn nhất có thể
- Shorten URL gồm: chữ số (0-9) và chữ cái (a-z, A-Z)
- Để đơn giản shorten URL sẽ không bao giờ được updated hoặc xoá

Những use-cases cơ bản:

- Long URL → Shorter URL
- Click vào shorten URL → redirect đến URL gốc
- High availability, scalability, fault tolerance

### Envelop estimation

- Write operation: 100 triệu URL trong ngày
- Write operation / s: 100 triệu / 24 / 3600 = 1160
- Read operation: giả sử tỉ lệ `read:write` là `10:1` ta sẽ có read operation / s = 1160 * 10 = 11,600
- Giả sử hệ thống hoạt động trong vòng 10 năm, do đó ta cần lưu trữ: `100 triệu * 365 * 10 = 365 tỉ records`
- Giả sử độ dài trung bình của URL là 100, vậy ta cần: `365 tỉ * 100 = 36.5TB` để lưu trữ

## Bước 2: High-level design

Ở bước này ta sẽ tiếp cận với 3 vấn đề: **API endpoints**, **URL redirecting**, **URL shortening flows**

### API Endpoint

Đây là điểm tương tác giữa client và hệ thống của chúng ta. Giả sử ta thiết kế hệ thống theo REST-style, ở đây ta cần 2 APIs

#### Shortening URL (POST) - api/v1/data/shorten

request parameter `{longURL: longURLString}`
return data: `shortURL`

#### URL redirecting (GET) - api/v1/shortURL

return data: `longURL` for redirectiion

### URL redirecting

Khi ta gõ shorten URL vào trình duyệt thì request sẽ có cấu trúc như hình dưới đây

![Screen Shot 2022-09-10 at 12 12 50](https://user-images.githubusercontent.com/15076665/189466722-fe958638-1331-475e-9341-fd676faf8288.png)

Ở đây `status code: 301` nhằm mục đích redirection. Quá trình giao tiếp giữa client và shorten URL server sẽ diễn ra như dưới đây:

![Screen Shot 2022-09-10 at 12 17 32](https://user-images.githubusercontent.com/15076665/189466738-351e9304-fd96-4431-bed3-8378bc2af439.png)

Giải thích một chút về sự khác biệt giữa `301` và `302` status

`301`: có thể hiểu là `permanently` redirect. Tức là response từ phía `shorten URL service` sẽ được cache lại và từ lần sau trở đi khi browser gửi đi shorten URL, nó sẽ được redirect luôn tới original URL mà không thông qua `shorten URL service`

`302`: có thể hiểu là `temporarily` redirect. Tức là response từ `shorten URL service` sẽ không được cache, mỗi lần khi browser gửi đi shorten URL thì nó sẽ phải gọi tới `shorten URL service` rồi mới redirect đến original URL.

Mỗi một loại redirection đều có ưu và nhược điểm riêng. Nếu chú trọng đến hiệu năng của hệ thống (giảm đi số lượng requests đến `shorten URL service`) thì `301` là sự lựa chọn phù hợp. Còn nếu chú trọng tới `analytics` để tracking số lần user click vào URL thì `302` sẽ thích hợp cho trường hợp này.

Cấu trúc dữ liệu sử dụng để lưu trữ `shortURL` và `longURL` là **hashTable** theo dạng dưới đây

```TS
hashTable<shortURL, longURL>
```

khi cần lấy về longURL ta sẽ làm như sau `hashTable.get(shortURL)`

### URL Shortening

Giả sử với shorten URL: `www.tinyurl.com/{hashValue}`. Ta cần tìm một **hash function** để map `longURL` với `hashValue` như ở URL trên

![Screen Shot 2022-09-10 at 12 31 03](https://user-images.githubusercontent.com/15076665/189467024-1ca4ba76-3a4a-4366-9e71-04b4da1f6970.png)

Có 2 yêu cầu với `hash function` như sau:

- Phải hash `longURL` thành `hashValue`
- Phải map ngược lại từ `hashValue` về `longURL`

## Bước 3: Thiết kế chi tiết

Ở bước này ta sẽ tập trung vào `Data model`, `Hash function`, `URL shortening`, `URL redirecting`

### Data model

Giải pháp sử dụng hash table chỉ áp dụng được khi số lượng records nhỏ. Trong thực tế số lượng records là rất lớn nên sử dụng memory là không đủ.

Do đó ta cần phải lưu trữ `shortURL` và `longURL` vào RDB như cấu trúc bảng "đơn giản" dưới đây:

![Screen Shot 2022-09-10 at 12 35 42](https://user-images.githubusercontent.com/15076665/189467161-6b1c0482-9f3e-4651-ab6d-24f4dc2f0441.png)

### Hash function

Ta cần tìm một hàm băm để map giữa `longURL` và `hashValue`. Giả sử với độ dài của hash value là `n`, mỗi một kí tự của hash value sẽ nằm trong tập [0-9a-zA-Z] → có cả thảy (10 + 26 + 26 = 62 kí tự) vậy nên như ở phần **Envelop estimation** ta cần lưu trữ 365 tỉ records ta thấy rằng điều kiện `62^n >= 365 tỉ` cần phải thoả mãn. Với `n = 7` ta có thể lưu trữ `62^7 = 3500 tỉ` records vậy nên ta có thể đưa ra kết luận các hash value sẽ có độ dài là 7

Về loại hàm băm ta có 2 loại:

- Hash + collision
- Base 62 conversion

#### Hash + collision

Ta cần hàm băm để "băm" longURL thành hash value có độ dài là 7. Các hàm băm phổ biến là `MD5`, `SHA-1`, `CRC32`

Các hàm băm này đều cho ra các giá trị băm với độ dài > 7. Cách tiếp cận ở đây đó là lấy ra 7 kí tự đầu tiên của hash value nhận được. Tuy nhiên điều này có thể dẫn đến tình trạng bị trùng lặp, do đó cách giải quyết ở đây đó là: nếu bị trùng lặp ta sẽ thêm các pre-defined string và băm (quá trình này có thể lặp đi lặp lại nhiều lần) như hình minh hoạ dưới đây:

![Screen Shot 2022-09-10 at 12 57 28](https://user-images.githubusercontent.com/15076665/189467763-a24e9f29-f00a-48a2-923e-8747de986d16.png)

Cách làm này có thể giải quyết được vấn đề trùng lặp shorten URL như ở trên, tuy nhiên việc query vào DB liên tục ở mỗi request sẽ tăng chi phí và giảm hiệu năng. Bloom filter có thể được cân nhắc ở đây như một giải pháp kiểm tra xem short URL đã được lưu hay chưa.

#### Base 62 conversion