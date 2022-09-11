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

Để gửi được notify, sms hay email thì thứ ta cần chính là: device token, phone number, mail address. Những việc này sẽ được thực hiện khi người dùng cài đặt app hoặc sign up vào hệ thống của chúng ta, khi đó thông tin liên tạc của người dùng sẽ được lưu trữ trong DB

![Screen Shot 2022-09-11 at 22 11 33](https://user-images.githubusercontent.com/15076665/189529427-0dbbb27d-dfb6-4e3d-aa5c-89d8b0e01c42.png)

Hình dưới đây sẽ mô tả việc lưu trữ `phone number`, `email`, `device token` trong DB như thế nào. `email` và `phone number` sẽ được lưu trong `bảng user` còn `device token` sẽ được lưu trong bảng `device` vì một user có thể có nhiều device và notify sẽ phải truyền tới mọi devices.

![Screen Shot 2022-09-11 at 22 13 48](https://user-images.githubusercontent.com/15076665/189529601-bfcc540c-4377-498b-8b11-01882cfc471b.png)

### Flow gửi/nhận notification

![Screen Shot 2022-09-11 at 22 17 03](https://user-images.githubusercontent.com/15076665/189529698-f4664271-49dd-40ca-b195-9bb2db46e5a9.png)

**Service 1 → N** có thể là micro-services hoặc cron-jobs, ... bất kì thứ gì có thể trigger việc gửi notify đến user

**Notification system** là trung tâm của hệ thống, cung cấp API cho **Service 1 → N** sử dụng, build payload cho third party service. Có thể bắt đầu đơn giản bằng 1 server duy nhất

**Third-party services** có trách nhiệm chuyển notification đến cho người dùng, hệ thống của chúng ta nên được thiết kế để có tính mở rộng cao. Tính mở rộng cao ở đây được hiểu theo hướng có thể thay thế service này bằng một service khác dễ dàng mà không làm ảnh hưởng đến toàn bộ hệ thống. Nguyên nhân có thể là service A không hoạt động ở một region nào đó chẳng hạn (FCM không hoạt động ở Trung Quốc)

**iOS, Android, SMS, Email** user nhận notification trên device của họ

Một vài vấn đề của thiết kế trên mà ta có thể kể ngay ra ở đây đó là:

- Single Point Of Failure (SPOF): do chỉ có 1 server duy nhất
- Khó để scale do hệ thống hiện đang hoạt động chỉ với 1 server, việc scale sẽ vấp phải các vấn đề như cache, DB, ...
- Bottle neck: do việc gửi payload đến Third-party service có thể sẽ mất thời gian để nhận về response

### Phiên bản cải thiện

Ta sẽ cải thiện hệ thống theo hướng sau:

- Đưa cache và DB ra khỏi notification server
- Thêm server và thiết lập auto horizontal scaling
- Sử dụng message queue để decouple các components trong hệ thống

![Screen Shot 2022-09-11 at 22 45 16](https://user-images.githubusercontent.com/15076665/189530860-75822eff-4364-4f5d-9088-6da89539799e.png)

Những điểm khác biệt của hệ thống cải thiện so với phiên bản cũ

**Notification server:**

- Cung cấp APIs cho các service gọi, các APIs này nên là internal API đế tránh spam
- Tiến hành basic validation như verify email, phone number
- Query data từ cache hoặc DB để build notification
- Đẩy dữ liệu về notification vào các message queue để tiến hành xử lí song song

Dưới đây là ví dụ về API send email

`POST: https://api.example.com/v/sms/send`

```JSON
{
  "to": [
    {
      "user_id": 265,
    },
  ],
  "from": {
    "email": "no-reply@mail.com",
  },
  "subject": "Hello",
  "content": [
    {
      "type": "text/plain",
      "value": "Test",
    },
  ],
}
```

**Cache**: user info, user device token, notification template đều sẽ được cache
**DB**: lưu dữ liệu về user, notification, setting, ...
