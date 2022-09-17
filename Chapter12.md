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

![Screen Shot 2022-09-16 at 8 14 56](https://user-images.githubusercontent.com/15076665/190525180-b8ba06b5-28e3-40f6-a230-58e2b129de6c.png)

Trong `long polling`, client sẽ giữ connection tồn tại cho đến khi nhận được message hoặc đạt tới ngưỡng timeout.

Khi client nhận được message, nó sẽ gửi req đến server để tiếp tục quá trình "long polling" như trên. Tuy nhiên bản thân `long polling` cũng có một vài nhược điểm như sau:

- Sender và receiver có thể không kết nối tới cùng một server. Nguyên nhân: HTTP là stateless (server sẽ không hề biết về trạng thái của client). Nếu sử dụng `round robin` cho `load balancer` thì server nhận được message **chưa chắc** đã kết nối với client cần nhận message
- Server không thể báo cho client rằng kết nối bị ngắt
- Trong trường hợp user không chat quá nhiều thì req vẫn được gửi đi liên tục khi timeout

### WebSocket

Đây là giải pháp phổ biến để server cập nhật các sự thay đổi cho client

![Screen Shot 2022-09-17 at 10 46 43](https://user-images.githubusercontent.com/15076665/190835781-49d79114-6b27-4df5-8d27-4b0b74d945f2.png)

WebSocket là một kết nối hai chiều (bi-directional) được khởi tạo bởi phía client. Khởi tạo thì nó vẫn là một `HTTP connection` và có thể được "upgrade" thông qua bước handshake.

Thông qua kết nối "bền vững" này server có thể gửi các updates cho client. Hơn nữa WebSocket connection sử dụng các cổng **80** và **443** giống như **HTTP** và **HTTPS** nên vẫn có thể hoạt động tốt ngay cả khi có **firewall**

Như đã thấy ở các phần trước, với sender thì **HTTP connection** là một sự lựa chọn tốt nhưng với **receiver** thì hoàn toàn không.

Do WebSocket connection là `persistent` nên việc quản lí connection sẽ được thực hiện hoàn toàn ở bên phía server.

### High-level design

Trong một hệ thống chat không nhất thiết phải sử dụng `stateful` toàn bộ, thay vào đó vẫn có những module sử dụng `stateless` ví dụ như: `Authentication service`, `Service discovery`, ...

Hình bên dưới sẽ mô tả tổng quan về hệ thống gồm: `Stateless`, `Stateful` và `Third-party` integration

![Screen Shot 2022-09-17 at 10 56 57](https://user-images.githubusercontent.com/15076665/190836109-d0780910-fd9a-4242-b3c7-dd92c205c302.png)

#### Stateless service

Vẫn theo hình thức "truyền thống" đó là `public-facing request/ response services`. Sẽ là `login`, `signup`, `user profile`, ...

Các `Stateless service` này sẽ nằm sau load balancer (nhiệm vụ và điều phối request) đến service chính xác dựa theo request paths.

Các services phía sau này có thể là:

- Monolithics
- Microservices

Một chú ý khác đó là `Service discovery` có nhiệm vụ tìm DNS của chat server mà client sẽ kết nối tới.

#### Stateful service

Stateful service duy nhất ở đây là `chat service`. Stateful là bởi vì client sẽ maintain persistent connection tới chat server.

Về cơ bản thì client sẽ không chuyển server cho đến khi server hiện tại không còn hoạt động nữa. Service discovery sẽ tìm ra chat server phù hợp để tránh tình trạng quá tải.

#### Third-party integration

Với một chat system thì `push notification` là một service đi kèm không thể thiếu vì nó sẽ là cách để thông báo cho user khi có message mới kể cả khi app không hoạt động.

### Scalability

Với thiết kế như hiện tại, ta hoàn toàn có thể đặt toàn bộ vào một server duy nhất. Yêu cầu của thiết kế đó là đáp ứng 1 triều user, mỗi user trung bình cần 10K memory để connect, do vậy ta cần cả thảy 10GB cho user connection.

Tuy nhiên việc đặt mọi thứ vào 1 server sẽ là một "điểm trừ" rất lớn trong mắt của interviewer. Sẽ có nhiều vấn đề xảy ra nếu đặt mọi thứ vào một server duy nhất, có thể kể ra tiêu biểu đó là **Single Point Of Failure**

![Screen Shot 2022-09-17 at 11 18 26](https://user-images.githubusercontent.com/15076665/190836816-3d84e013-7a6c-4dc9-b304-a1fac4d29422.png)

**Presence server** quản lí online/ offline status
**Key-Value store** được sử dụng để lưu lịch sử chat, khi user chuyển từ trạng thái offline sang online, user sẽ thấy toàn bộ lịch sử chat của mình.

### Storage

Việc đưa ra các quyết định liên quan đến data layer sẽ mất nhiều thời gian và công sức hơn bình thường. Ở đây chúng ta sẽ cân nhắc sự lựa chọn giữa RDB và NoSQL, để đưa ra sự lựa chọn ta cần xem xét đến 2 yếu tố sau:

- Data types
- Read/ Write patterns

#### Data types

Có 2 loại data type trong một chat system

- Generic data: đây là dữ liệu liên quan đến user profile, setting, ...Các dữ liệu kiểu này sẽ được lưu trong RDB. Các kĩ thuật như sharding hay replication sẽ được sử dụng để phục vụ cho yêu cầu về **availability** và **scalability**
- Chat history data:  với kiểu dữ liệu này ta cần nắm rõ về `read/ write pattern`
① Số lượng chat data là rất lớn (Facebook messenger xử lí 60 tỉ messages / ngày)
② User thường chỉ xem các chat messages gần đây
③ Vẫn có trường hợp user xem lại các messages trong quá khứ như: search message, jump đến một message cụ thể nào đó, view mention, ...
④ Tỉ lệ đọc / viết là 1 : 1

**Key-value store** được khuyên dùng ở đây với những lí do sau:

- Horizontall scaling dễ dàng
- Low latency khi truy cập dữ liệu
- RDB không xử lí long tail data tốt, khi index tăng thì random access sẽ tốn chi phí
- Các ứng dụng chat nổi tiếng như Discord hay Facebook messenger đều sử dụng Key-value store

### Data models

#### Message table cho 1 on 1 chat

Primary key ở đây chính là `message_id`, id này sẽ giúp chúng ta quyết định thứ tự của message, không thể dựa vào `created_at` được vì thuộc tính này có thể trùng nhau về mặt giá trị đối với một vài messages.

![Screen Shot 2022-09-17 at 12 00 19](https://user-images.githubusercontent.com/15076665/190838169-206f9715-bc82-48a0-9e8b-e6e29a21148a.png)

#### Message table cho group chat

![Screen Shot 2022-09-17 at 12 11 56](https://user-images.githubusercontent.com/15076665/190838448-a0b5f5ed-a9f2-4d2c-a0d1-98b548f6118b.png)

Ở đây ta tạo `composite key` giữa `message_id` và `channel_id` (channel tương đương với group)

#### Message ID

Generate ID là một chủ đề khá hay ở đây. Message_id có nhiệm vụ đảm bảo thứ tự của các messages. Để làm được điều đó message id cần đáp ứng được 2 yêu cầu sau:

- IDs phải là duy nhất
- IDs có thể sắp xếp theo thời gian, tức là các rows mới sẽ có IDs cao hơn các rows cũ

Với điều kiện thứ 2, chúng ta thường nghĩ ngay tới `auto_increment` trong MySQL tuy nhiên `NoSQL` không hỗ trợ tính năng này.

Cách tiếp cận thứ 2 đó là sử dụng `global 64-bit sequence number generator` như `Snowflake`.

Cuối cùng là sử dụng `local sequence number generator`. Local ở đây nghĩa là mang tính cục bộ của một group / channel. Cách tiếp cận này đơn giản hơn so với `global ID implementation`

## Bước 3: Thiết kế chi tiết

Với chat system thì:

- Service discovery
- Messaging flows
- Online / offline indicator

sẽ là những phần mà chúng ta cần phải đi sâu ở đây

### Service discovery

Có chức năng đưa ra server

- Gần client nhất về mặt vị trí
- Có capacity đủ để đáp ứng yêu cầu của client

`Apache Zookeeper` là một open-source service discovery phổ biến

![Screen Shot 2022-09-17 at 12 22 36](https://user-images.githubusercontent.com/15076665/190838703-575973b6-f779-482e-af63-2fb29fe71986.png)

Ở đây Service discovery sẽ tìm ra server phù hợp cho client, trả về thông tin server cho client để client kết nối tới server thông qua web socket

### Message flows

#### 1 on 1 chat flow

![Screen Shot 2022-09-17 at 12 35 33](https://user-images.githubusercontent.com/15076665/190839024-3eecfb2a-48b4-4d22-83a3-be0e4f18063f.png)

Sau khi user gửi message đến `chat server 1`, chat server sẽ lấy được message ID từ ID generator.

Message sẽ được chat server 1 gửi tới `message sync queue`, message cũng được lưu lại trong key-value store

Nếu user B online thì message sẽ được forware đến chat server 2 (user B đang connect tới server này). Nếu offline thì push notification sẽ được gửi từ `Push notification server`.

Ở đây websocket connection sẽ được thiết lập giữa user B và chat server 2

#### Message synchrnoization giữa các devices

![Screen Shot 2022-09-17 at 12 42 24](https://user-images.githubusercontent.com/15076665/190839222-c72fe181-06aa-4513-97fe-a04abbb49f6e.png)

Ta thấy rằng khi user A login vào phone device, websocket connection sẽ được thiết lập giữa chat server và phone device **và cả laptop**.

Mỗi thiết bị sẽ có một biến gọi là *cur_max_message_id*, biến này sẽ lưu max message ID (ID của message mới nhất) trên thiết bị đó.

Các messages thoả mãn hai điều kiện dưới đây sẽ được xem như message mới:

- recipient ID = login user ID
- message ID trong key-value store phải lớn hơn *cur_max_message_id*

Việc sử dụng *cur_max_message_id* sẽ giúp quá trình đồng bộ các message mới trên từng thiết bị trở nên dễ dàng hơn

#### Small group chat flow

So với 1 on 1 chat thì group chat sẽ phức tạp hơn

![Screen Shot 2022-09-17 at 15 23 29](https://user-images.githubusercontent.com/15076665/190843710-ea0c45bb-0c41-4adb-82fe-09a0012084fe.png)

Như ở hình trên ta giả sử group có 3 members là A, B và C.

Khi user A gửi message, bản copy của message này sẽ được đưa vào `message queue sync` của 2 users B, C. Các `message queues` này có thể hiểu như `inbox` của user. Sử dụng cách này có 2 ưu điểm như dưới đây:

- Đơn giản để sync user message khi user chỉ cần "check inbox" là đủ
- Khi group member ít thì việc lưu bản sao của message ở nhiều message queues cũng không gây tốn bộ nhớ quá nhiều

WeChat cũng sử dụng cách tiếp cận tương tự trên, nên do đó họ giới hạn số member tối đa trong 1 group là 500.

Về cơ bản khi càng có nhiều member trong group thì số lượng bản sao message được lưu sẽ tăng lên.

Ở phía người nhận thì họ có thể nhận message từ nhiều senders khác nhau như hình minh hoạ dưới đây:

![Screen Shot 2022-09-17 at 15 35 42](https://user-images.githubusercontent.com/15076665/190843999-a3b84dd0-9f63-4c23-a9b4-32937a36e9a1.png)

#### Online presence

Dấu hiệu online màu xanh lam ở phía dưới profile picture chính là trang thái online/ offline của người dùng.

Về cơ bản `presence server` sẽ quản lí online status và tương tác với client thông qua `WebSocket`.

Dưới đây là high-level flow

**User login:**

Sau khi WebSocket connection được thiết lập giữa client và real-time service, user A online status và *last_active_at* time-stamp sẽ được lưu trong `key-value store`. Presence indicator sẽ show user đang online sau khi login

![Screen Shot 2022-09-17 at 15 41 41](https://user-images.githubusercontent.com/15076665/190844224-431c4e67-2709-44a2-a9c4-558d6c786edd.png)

**User logout:**

Khi user logout thì `status` trong `key-value store` sẽ chuyển thành `offline` và `presence indicator` sẽ show user đã offline

![Screen Shot 2022-09-17 at 16 03 13](https://user-images.githubusercontent.com/15076665/190844978-8f55bf79-2be9-486a-8b46-526200011254.png)

**User disconnect:**

Trong trường hợp user ngắt kết nối khỏi internet thì cách tiếp cận update `key-value store` thành `offline` và update lại thành `online` khi user kết nối trở lại sẽ không đem lại UX tốt. Nguyên nhân bởi việc kết nối của user bị chập chờn hoàn toàn có thể xảy ra thường xuyên, hơn thế nữa khi user vào tunnel thì user sẽ bị ngắt kết nối tạm thời khỏi internet.

Ta sẽ áp dụng cơ chế `hearbeat` để giải quyết vấn đề này. Trong một khoảng thời gian là **x giây** user sẽ gửi `heartbeat event` cho `presence server`. Nếu server nhận được thì user sẽ được xem như online và ngược lại sẽ là offline.

Hình dưới là minh hoạ cho cơ chế `heartbeat` với period time là 5s. Sau 30s khi server không nhận được `heartbeat event` thì user sẽ được xem như là đang offline.

![Screen Shot 2022-09-17 at 16 12 41](https://user-images.githubusercontent.com/15076665/190845234-687d2931-113b-4682-ab1c-9fb18273cbf5.png)

#### Online status fanout

Vậy làm thế nào để các user khác biết được việc user A chuyển online / offline status. Câu trả lời đó là mô hình `publisher - subscriber`

![Screen Shot 2022-09-17 at 16 20 25](https://user-images.githubusercontent.com/15076665/190845543-55a51928-8936-4137-9d5a-40fa3296f117.png)

Hình phía trên mình hoạ cho mô hình `publisher - subscriber` của `presence server`. Giả sử ngoài user A còn có 3 users khác là B, C, D. Server sẽ maintain các friend pair channels (A-B, A-C, A-D). Khi user online status thay đổi sẽ có một event được đưa vào channel và các user B, C, D khi subscribe channel sẽ nhận được việc thay đổi online status của user A.

Tương tác giữa các clients với server đều thông qua real-time websocket.

Thiết kế này được `WeChat` sử dụng và chỉ áp dụng cho một nhóm nhỏ các user. Khi số lượng user lớn thì thiết kế này sẽ tốn thời gian và chi phí.

Với group có 100,000 users thì mỗi khi user thay đổi status thì sẽ phát sinh ra 100,000 events. Cách giải quyết bottleneck performance đó là chỉ update status khi user vào group hoặt refresh friend list

## Bước 4: Tổng quát

Một vài điểm sau đây nên được thảo luận với interviewer nếu có thời gian:

- Hỗ trợ gửi file, ảnh, video. Nén dữ liệu, cloud storage, thumbnails là các chủ đề khá hay
- End-to-end encryption. Whatsapp hỗ trợ end-to-end encryption, chỉ có sender và các recipient mới đọc được message
- Caching message phía client sẽ giúp giảm đi lượng dữ liệu trao đổi giữa server và client
- Error handling:
  - Khi Chat server gặp sự cố, service discovery (Zookeeper) sẽ tìm ra server phù hợp để client connect tới
  - Message resent mechanism. Retry và queueing là các kĩ thuật phổ biến để giải quyết vấn đề này
