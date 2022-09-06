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
