# Chương 9: Thiết kế một web crawler

Đây là một dạng câu hỏi phỏng vấn hết sức cơ bản và kinh điển trong các bài phỏng vấn về System Design.

Một web crawler sẽ được sử dụng cho những mục đích dưới đây:

- Search engine indexing: điều này được các web search engine triển khai (VD: Googlebot cho Google search)
- Web archiving (lưu trữ web): thu thập thông tin từ các trang web để dùng cho tương lai
- Web mining: phân tích các thông tin có trên web nhằm phục vụ cho mục đích data mining
- Web monitoring

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Thuật toán của các hệ thống web crawler về cơ bản gồm 3 bước sau:

1. Với tập các URLs, download các web page được định nghĩa bởi các URLs đó
2. Bóc tách các URLs bên trong các pages trên
3. Thêm các URLs mới vào list các URLs và lặp lại từ bước 1

Dưới đây là các yêu cầu ta cần xác nhận:

- Web crawler dùng cho mục đích gì ? Ví dụ: Search engine indexing
- Mỗi tháng sẽ có 1 tỉ pages được thu thập
- Nội dung của page chỉ là thuần HTML
- Các yếu tố như edit hoặc thêm mới pages cần phải được cân nhắc
- Hệ thống cần phải hoạt động liên tục trong 5 năm
- Các pages có nội dung trùng nhau sẽ bị loại bỏ

Ngoài ra ta cũng nên tự liệt kê ra những yếu tố quan trọng của một hệ thống web crawler tốt như sau:

- Robustness: Xử lí được các edge cases (bad URL, malicious links)
- Politeness: Hệ thống Crawler không nên tạo quá nhiều request tới web page trong một khoảng thời gian ngắn
- Extensibility: Hệ thống cần được thiết kế để có khả năng mở rộng cao, tránh tình trạng phải re-design khi requirement thay đổi

### Back of the envelope estimation

- Giả sử 1 tỉ pages được download mỗi tháng
- QPS = 1,000,000,000 / 30 / 24 / 3600 =~ 400 pages/s
- Peak QPS = 2 * QPS = 800
- Giả sử size trung bình của 1 page là 500k
- Mỗi tháng cần lưu trữ: `1 tỉ * 500 = 500TB`
- Trong 5 năm cần lưu trữ: `500 * 12 * 5 = 30PB`

## Bước 2: High level design

Thiết kế tổng quan cho hệ thống web crawler sẽ như dưới đây:

![Screen Shot 2022-09-11 at 13 14 06](https://user-images.githubusercontent.com/15076665/189512116-a9fee6d6-b437-4c78-9cde-6e9b972e861c.png)

### Seed URLs

Đây chính là điểm đầu vào của một hệ thống crawler. Ta lấy ví dụ về việc crawl dữ liệu web của một trường ĐH, cách làm đơn giản nhất ở đây đó là dựa theo domain để tạo `seed URLs`

Việc tạo seed URLs đóng vai trò quan trọng trong việc ta có thể quét được nhiều link nhất có thể hay không. Một phương pháp hiệu quả trong việc tạo seed URLs chính là việc chia nhỏ URL space, có thể là:

- Web site phổ biến tuỳ theo quốc gia, lãnh thổ
- Theo từng chủ đề: shopping, thể thao, ...

### URL Frontier

Các web crawler hiện tại thường chia crawl states thành 2:

- Sẽ được downloaded
- Đã được downloaded

URL frontier sẽ lưu trữ các URL sẽ được downloaded, nó có cấu trúc tương tự như một FIFO queue

### HTML Downloader

Download HTML page content dựa theo các URL được cung cấp bởi `URL Frontier`

### DNS Resolver

`HTML Downloader` sẽ gọi đến `DNS Resolver` để chuyển domain name thành IP address cho mục đích download page.

VD: www.wikipedia.com → 198.35.26.96

### Content Parser

Nội dung được download về cần phải được kiểm tra đế tránh các nội dung xấu cũng như việc lưu trữ lãng phí.

Tuy nhiên triển khai content parser bên trong crawling server sẽ làm giảm đi tốc độ crawl của hệ thống nên `content parse sẽ được triển khai như một component riêng`.

### Content seen ?

Theo nghiên cứu thì có đến 29% nội dung của các trang web bị trùng lặp nên việc lưu trữ cần được xem xét để tránh lãng phí tài nguyên đặc biệt khi hệ thống phải xử lí cả tỉ web pages.

`Content seen data structure` được sử dụng để kiểm tra xem nội dung page đã được lưu trữ hay chưa.

Một phương pháp đơn giản được sử dụng đó là so sánh nội dung các HTML page theo từng kí tự (character by character) tuy nhiên cách so sánh này khá mất thời gian nên ta sẽ so sánh `hash value` của các HTML page content.

### Content Storage

Việc lựa chọn cách thức lưu trữ sẽ phụ thuộc vào:

- Data type
- Data size
- Access frequency
- ...

Thông thường thì cả `disk` và `memory` sẽ được sử dụng. Do nội dung cần lưu trữ rất lớn nên:

- Hầu hết web page content sẽ được lưu trữ ở disk
- Các nội dung thường dùng sẽ được lưu trữ ở memory

### URL extractor

Component này có nhiệm vụ bóc tách các URLs có trong page, chuyển chúng từ relative path thành absolute path

![Screen Shot 2022-09-11 at 13 51 28](https://user-images.githubusercontent.com/15076665/189512978-c5896d52-c802-4356-99d6-5adb2b33d8aa.png)

### URL filter

Lọc các URLs trong "blackedlist"

### URL Seen ?

Là data structure kiểm tra xem URL đã được duyệt hay chưa hoặc đang có trong URL Frontier hay không.

Việc làm này giúp giảm đi "gánh nặng" cho server cũng như tránh các `infinite loop`

### URL Storage

Lưu trữ các URLs đã duyệt

## Bước 3: Thiết kế chi tiết

Trong phần này ta sẽ bàn về những thành phần sau khi tiến hành thiết kế chi tiết

- DFS vs BFS
- URL Frontier
- HTML Downloader
- Robustness
- Extensibility
- Xử lí các vấn đề liên quan tới nội dung page

### DFS vs BFS

Ta có thể coi hệ thống web giống như một đồ thị với:

- Node là các pages
- Cạnh của đồ thị sẽ là các URLs

DFS duyệt đồ thị theo chiều sâu "rất sâu" nên ta sẽ sử dụng BFS ở đây. BFS triển khai bằng việc sử dụng FIFO queue. Tuy nhiên 2 vấn đề dưới đây sẽ phát sinh:

- Hầu như các links từ một page đều là `internal-link`, việc quét qua các link của một web quá nhiều lần có thể làm tăng tải của server. Vấn đề này được gọi là "impolite"
- Các web pages luôn có những độ ưu tiên và độ quan trọng khác nhau về mặt nội dung, do dó việc cần làm ở đây đó là đánh độ ưu tiên cho các URLs

### URL frontier

Component này sẽ giúp giải quyết hai vấn đề nêu trên. Đây là một data structure đảm nhận:

- Store URLs sẽ được download
- Đảm bảo độ ưu tiên của các URLs
- Politeness
- Freshness

### Politeness
