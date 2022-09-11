# Chương 10: Thiết kế notification system

Notification system có phạm vi rộng hơn mobile push notification. Nó bao gồm:

- Mobile push notification
- SMS message
- Email

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Các câu hỏi khi phỏng vấn thường có độ mở cao và có những yêu cầu không thực sự rõ ràng. Do đó một trong những điều cần làm ở đây đó là hỏi interviewer những câu hỏi giúp bạn có thể nắm bắt rõ hơn về yêu cầu của hệ thống

Ở bài toán thiết kế này ta cần:

- Hệ thống có khả năng push notification, SMS message, email
- Soft real-time system, notification nên đến được user càng sớm càng tốt, nếu hệ thống có nhiều users thì một chút delay vẫn có thể chấp nhận được
- Cần hỗ trợ iOS/ android devices và laptop/ desktop
- Notifications có thể được triggered bởi client app hoặc scheduled ở phía server
- User có thể chọn option không nhận notification (opt-out)
- Một ngày sẽ có: 10 triệu mobile push notification, 1 triệu SMS message, 5 triệu emails

## Bước 2: High-level design

Ta cần xem xét những cấu trúc sau:

- Các loại notification khác nhau
- Flow thu thập thông tin liên lạc
- Flow gửi/nhận notification

### Các loại notification khác nhau

#### iOS push notification

![Screen Shot 2022-09-11 at 18 44 52](https://user-images.githubusercontent.com/15076665/189521208-e092c43d-986f-4493-8e6e-d3fa16c3950a.png)

Để thực hiện push notification cho các iOS devices ta cần 3 thành phần như sau:

- Provider: build và gửi notification req đến Apple Push Notification Service (APNs), provider cần cung cấp các thông tin như sau:
① Device token: unique identifier sử dụng để push notification
② Payload: JSON data bao gồm thông tin về notification payload, ví dụ như sau:

```JSON
{
  "aps": {
    "alert": {
      "title": "Game Request",
      "body": "Bob wants to play chess",
    },
    "badge": 5,
  },
}
```

- APNs: Remote service được cung cấp bởi Apple với nhiệm vụ "lan truyền" notification đến iOS devices
- iOS device: client nhận notify

#### Android push notification

Flow hoàn toàn tương tự như iOS nhưng khác ở chỗ thay vì sử dụng APNs thì Android sử dụng FCM (Firebase Cloud Message)

![Screen Shot 2022-09-11 at 18 52 23](https://user-images.githubusercontent.com/15076665/189521454-041cf5df-4e8f-43a0-9c7c-b45f0ece967e.png)

#### SMS message

Với SMS ta sử dụng các service "kinh điển" như **Twilio**, **Nextmo**

![Screen Shot 2022-09-11 at 18 53 39](https://user-images.githubusercontent.com/15076665/189521517-922223dc-8d62-478a-9cfd-f15f19e35784.png)

#### Email

Mỗi công ty có thể setup riêng một mail server cho mình tuy nhiên để đảm bảo các vấn đề về hiệu năng hay tốc độ thì các email server phổ biến như "Sendgrid" có thể được sử dụng ở đây

![Screen Shot 2022-09-11 at 18 55 12](https://user-images.githubusercontent.com/15076665/189521586-c88d756a-3da6-45d3-9b68-5e1ac4a84d4d.png)

Hình dưới đây sẽ là kết hợp cả 4 loại notification trên

![Screen Shot 2022-09-11 at 18 56 48](https://user-images.githubusercontent.com/15076665/189521624-079b6bd7-cd33-48bd-b5c5-d1f4d4a6d6ab.png)

### Thu thập thông tin liên lạc
