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
