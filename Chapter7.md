# Chương 7: Thiết kế unique ID generator trong hệ thống phân tán

Với một hệ thống phân tán với nhiều databases khác nhau thì việc sử dụng `auto_increment` hoàn toàn không đem lại hiệu quả gì.

## Bước 1: Hiểu vấn đề và nhận định design scope

ID phải:

- Unique
- Sortable

IDs được tạo ra với số lượng nhiều hơn vào buổi tối
ID chỉ bao gồm **chữ số** và có độ dài 64-bits. Hệ thống cần phải có khả năng generate 10,000 IDs/s

## Bước 2: High-level design

Có nhiều lựa chọn để câ nhắc ở đây:

- Multi-master replication
- UUID: Universal unique identifier
- Ticket server
- Twitter snowflake approach

### Multi-master replication

Cách tiếp cận này sử dụng `auto_increment` của database nhưng thay vì `+1` theo thời gian, ID sẽ `+k` với k là số lượng database servers

Như hình minh hoạ bên dưới đây các IDs trong từng database server sẽ được `+2`

![Screen Shot 2022-09-06 at 23 41 40](https://user-images.githubusercontent.com/15076665/188754893-8a476a03-a719-4b52-a8b1-719e6faac248.png)

Tuy nhiên sẽ gặp phải 2 vấn đề sau:

- Khó triển khai khi scale trên nhiều data center
- Scale không tốt khi tăng hoặc giảm database servers

### UUID

UUID sử dụng 128-bits phục vụ cho identify information.
Bản thân UUID cũng rất khó bị trùng lặp

VD về UUID như sau: `09c9awd2a-2121dn1d1d-12d12d1f13q`. UUID có thể được generate độc lập mà không cần đến coordinate giữa các servers

![Screen Shot 2022-09-07 at 8 31 01](https://user-images.githubusercontent.com/15076665/188757783-98982531-d1f9-45ff-9c41-5bf4552ae431.png)

Như hình trên mỗi server sẽ có một ID generator độc lập. Việc gen ID của mỗi server không hề liên quan đến server khác

**Ưu điểm:**

- Triển khai đơn giản và dễ dàng scale
- Tránh được vấn đề về đồng bộ hoá giữa các servers

**Nhược điểm:**

- 128-bits, dài hơn yêu cầu hệ thống là 64-bits
- Chứa cả chữ trong khi yêu cầu chỉ chứa chữ số

### Ticket Server

Đây là phương pháp generate ID dùng cho các hệ thống phân tán

![Screen Shot 2022-09-07 at 23 00 11](https://user-images.githubusercontent.com/15076665/188897535-673bd856-433f-43fe-a5d9-e456609efde9.png)

Về cơ bản phương pháp này dựa theo `auto_increment` của database server duy nhất

**Ưu điểm:**

- Dễ triển khai với các hệ thống quy mô vừa và nhỏ
- Numeric IDs

**Nhược điểm:**

- **Single Point Of Failure**: khi database server "sập" thì các modules phụ thuộc vào nó cũng sẽ bị ảnh hưởng theo, để tránh điều này ta có thể triển khai multiple database servers tuy nhiên lúc này ta sẽ phải đối mặt với vấn đề đồng bộ hoá dữ liệu giữa các servers

### Twitter Snowflake approach

Bản chất ở đây đó là "chia để trị". Ta tiến hành chia nhỏ ID thành nhiều sections con.

![Screen Shot 2022-09-07 at 23 07 48](https://user-images.githubusercontent.com/15076665/188899904-8f7ab1ae-db2a-4c40-b492-708ff0154e67.png)

① Bit dấu (1 bit): luôn là 0 (dùng cho tương lai sau này để phân biệt sign và unsign ID)
② Timestamp (41 bits): timestamp, tính bằng milisecond từ thời điểm hiện tại cho đến `epoch` hoặc `custom epoch`. Giá trị mặc định của `Twitter Snowflake epoch` là 1288834974657 (~ Nov 04, 2010 01:42:54 UTC)
③ Datacenter ID (5 bits): cho phép ta có thể có tất cả 2^5 = 32 data centers
④ Machine ID (5 bits): cho phép 2^5 = 32 máy trong một data center
⑤ Sequence Number (12 bits): increment 1 với mỗi ID được tạo mới. Được reset về 0 mỗi milisecond.

## Bước 3: Deep-dive design

Thông thường `Datacenter ID` và `Machine ID` sẽ được fix cứng khi ID generator bắt đầu hoạt động. Việc thay đổi các bit thuộc 2 sections này cần được cân nhắc cẩn thận để tránh việc conflict IDs.

Timestamp và sequence number sẽ được generated thường xuyên khi generator hoạt động

**Timestamp** sẽ tăng dần theo thời gian và các IDs sẽ được sắp xếp theo thứ tự thời gian
