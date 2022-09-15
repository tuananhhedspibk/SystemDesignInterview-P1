# Chuơng 12: Thiết kế chat system

## Bước 1: Xác định yêu cầu và phạm vi thiết kế

Một hệ thống chat one-on-one và group-chat là hoàn toàn khác nhau về mặt thiết kế do đó trước khi bắt tay vào việc thiết kế, bạn cần phải hỏi rõ interviewer về việc hệ thống chat này là one-on-one hay group.

Dưới đây là một số điều cần làm rõ (chú thích, phía dưới đây tôi trình bày theo format `câu hỏi ? (câu trả lời)`):

- Hệ thống chat thuộc loại nào ? (one-on-one và group-chat)
- Support những nền tảng nào ? (web và mobile)
- Scale của hệ thống ? (50 triệu DAU)
- Group-chat hỗ trợ tối đa bao nhiêu user một group ? (tối đa 100 người)
- Tính năng quan trọng là gì ? Có hỗ trợ attachment hay không ? (1 on 1, group chat, online indicator, chỉ hỗ trợ text message)
- Message size limit ? (100,000 kí tự tối đa)
- Chat story nên được lưu trữ trong vòng bao lâu ? (Mãi mãi)

Trong chương này chúng ta sẽ thiết kế một chat system giống như Facebook messenger:

- one-on-one chat với low latency
- Tối đa 100 người / group chat
- Online presence
- Multiple devices support (1 account có thể log in trên nhiều devices khác nhau)
- Push notification
- Hỗ trợ 50 triệu DAU

## Bước 2: High-level design

Điều quan trọng để có một thiết kế tổng quan tốt ở đây đó là cần phải nắm được cách giao tiếp giữa client và server. Về cơ bản client có thể là `mobile app` hoặc `web app`. Các clients sẽ không tương tác trực tiếp với nhau mà sẽ thông qua `chat service` hỗ trợ các tính năng như sau:

- Nhận mess từ các clients khác
- Tìm và gửi mess đến đúng địa chỉ
- Nếu user offline, server sẽ lưu chat message cho đến khi user online thì sẽ gửi đến cho user

Hình dưới đây sẽ mô tả cách thức giao tiếp giữa các clients (sender, receiver) với chat service

![Screen Shot 2022-09-15 at 23 14 48](https://user-images.githubusercontent.com/15076665/190427190-a2b06e20-415a-40a7-aeaa-aaf04d608c31.png)

Khi client khởi động một tiến trình chat, client sẽ kết nối tới `chat service` bằng một hoặc nhiều protocol khác nhau (việc nắm được các protocols này là rất quan trọng)

Như ví dụ ở trên, req được khởi tạo phía client và được gửi đến server thông qua HTTP protocol. Client gửi message đến cho service, yêu cầu service gửi message đến cho receiver.

`keep-alive` rất hữu dụng trong trường hợp này, nguyên nhân là do `keep-alive` giúp duy trì lâu dài kết nối giữa client và service qua đó tránh việc `TCP hand shake` xảy ra thường xuyên.

HTTP protocol rất ổn cho sender khi gửi mess khởi tạo (facebook messenger cũng sử dụng HTTP protocol)

Tuy nhiên việc **message được gửi từ server** thì lại phức tạp hơn. Nó có thể là các protocols khác như `polling`, `long-polling` hay `web socket`

### Polling

Như mô tả bên dưới, với `polling` thì client sẽ liên tục "hỏi" server về việc có message mới hay không. Việc "liên tục" này là tuỳ vào thiết lập tần suất của `polling`. Tuy nhiên trong nhiều trường hợp khi câu trả lời là "không" liên tục thì việc sử dụng polling sẽ gây lãng phí ít nhiều tài nguyên.

![Screen Shot 2022-09-15 at 23 26 56](https://user-images.githubusercontent.com/15076665/190430305-af1c63c2-4105-47d2-871e-6a34f8ba594b.png)

### Long polling

Do `polling` hoạt động không hiệu quả nên chúng ta sẽ xem xét đến `long polling` như dưới đây

![Screen Shot 2022-09-16 at 8 14 56] (https://user-images.githubusercontent.com/15076665/190525180-b8ba06b5-28e3-40f6-a230-58e2b129de6c.png)

Trong `long polling`, client sẽ giữ connection tồn tại cho đến khi nhận được message hoặc đạt tới ngưỡng timeout.

Khi client nhận được message, nó sẽ gửi req đến server để tiếp tục quá trình "long polling" như trên

