# Chuơng 11: Thiết kế news feed system

## Bước 1: Hiểu yêu cầu và phạm vi thiết kế

- Cần phát thiết kế cho cả web và mobile app
- User có thể publish post và nhìn thấy post của bạn bè trên news feed
- Các posts được sắp xếp theo thứ tự ngược thời gian (mới đến cũ)
- User có thể có 5000 bạn bè
- 10 triệu DAU
- Feed bao gồm image và video

## Bước 2: High level design

Thiết kế sẽ được chia thành 2 flow: `feed publishing` và `newsfeed building`

- **Feed publishing**: khi user publish feed, nó sẽ được lưu vào cache hoặc DB và sau đó sẽ được truyền tải đến news feed của bạn bè
- **Newsfeed building**: để đơn giản, ta giả sử rằng, newsfeed sẽ là tập hợp các feeds của bạn bè được sắp xếp theo trình tự thời gian gần nhất đến xa nhất

### Newsfeed APIs

Ở đây ta đề cập đến 2 APIs quan trọng nhất đó là:

#### Feed publishing API

`POST /v1/me/feed`

params:

- content: nội dung của post
- auth_token: dùng để xác thực API req

#### Newsfeed retrieval API

`GET /v1/me/feed`

params:

- auth_token: dùng để xác thực API req

![Screen Shot 2022-09-14 at 8 25 34](https://user-images.githubusercontent.com/15076665/190026722-ae5e7d2b-0f54-4657-abb2-cb595e8314af.png)

- Post service: lưu post trong cache và DB
- Fanout service: push new content vào news feed của bạn bè. Newsfeed data được lưu trong cache để tiện truy xuất
- Notification service: thông báo cho bạn bè biết nội dung mới, gửi đi notification

### News feed building

Dưới đây là high-level design

![Screen Shot 2022-09-14 at 22 58 46](https://user-images.githubusercontent.com/15076665/190174596-422f580e-49a7-4f46-96bc-bbf684ab5276.png)

- Newsfeed service sẽ fetch data từ cache
- Newsfeed cache sẽ lưu ID của news sẽ được render dưới client

## Bước 3: Deep dive design

### Feed publishing deep dive

Trong phần này ta sẽ đi sâu vào 2 phần: **web server** và **fanout service**

![Screen Shot 2022-09-14 at 23 03 05](https://user-images.githubusercontent.com/15076665/190175790-aca215b2-d5aa-45d7-a6fc-569394abdb45.png)

#### Web server

Ngoài việc tương tác với client, web server cần đảm bảo authen user khi user tiến hành tạo post thông qua `auth_token`. Hơn nữa `rate limiting` giúp hạn chế số lượng post mà user có thể tạo trong một khoảng thời gian nhất định để tránh spam.

#### Fanoute service

Đảm nhận nhiệm vụ truyền tải post đến cho các users khác. Có 2 mô hình cho việc quảng bá post này, đó là:

- Fanout on write (push model)
- Fanout on read (pull model)

#### Fanout on write

Ở cách tiếp cận này, new post sẽ được `pre-computed` tại thời điểm nó được tạo, new post sẽ được truyền tải đến cache của bạn bè ngay khi nó được tạo.

**Ưu điểm:**

- New post được generate real-time và được truyền tải đến bạn bè ngay lập tức
- Fetching news feed sẽ nhanh vì news feed được `pre-computed` ngay khi được tạo ra

**Nhược điểm:**

- Nếu user có nhiều bạn bè thì việc `pre-computed` cũng như real-time pushing sẽ tốn thời gian. Đây gọi là `hotkey problem`
- Với những user không thường xuyên active, việc `pre-computed` như trên là không cần thiết và gây tốn tài nguyên

#### Fanout on read

Cách tiếp cận ở đây đó là các post được tạo gần đây sẽ được pull khi user tiến hành tải trang

**Ưu điểm:**

- Cách làm này không gây lãng phí tài nguyên với các user không hoạt động thường xuyên
- Không gây ra `hotkey problem`

**Nhược điểm:**

- Fetching news feed sẽ chậm do news feed không được `pre-computed`

Chúng ta sẽ sử dụng cách làm hybrid. Tức là

- Sử dụng `Fanout on write` đối với những user active
- Sử dụng `Fanout on read` đối với user có nhiều followers hoặc friends để tránh lãng phí tài nguyên khi load page

`Consistent hashing` sẽ được cân nhắc nhằm giảm đi `hotkey problem` - phân tán việc pushing cho nhiều server

![Screen Shot 2022-09-14 at 23 16 33](https://user-images.githubusercontent.com/15076665/190179415-bb7b4e5c-a4ed-41a7-b19d-ff38a5950a06.png)

1. Fetch user id từ Graph DB, Graph DB là sự lựa chọn phù hợp cho việc lưu trữ các mối quan hệ `following/ follower`, `recommendation user`
2. Fetch user info dựa theo `user id` và `user setting` được lưu trong cache / DB. Hệ thống sẽ tiến hành filter các users sẽ nhận được post vì có những user đã block bạn hoặc bạn chỉ muốn một số users nhất định xem post mà thôi
3. Đưa friend list và post ID vào `message queue`
4. Các worker sẽ pull dữ liệu `<post_id, user_id>` từ `message queue` về. Việc lưu trữ dưới format `<post_id, user_id>` (hash table) sẽ giúp giảm đi lượng thông tin cần lưu trữ (tránh tốn tài nguyên bộ nhớ). Ngoài ra việc duyệt qua hash table chỉ gồm `post id` và `user id` cũng sẽ giúp hiệu ứng load posts phía client "mượt" hơn. Hơn nữa user chỉ quan tâm đến các post gần đây nên việc lưu trữ theo format nêu trên cũng giúp giảm tỉ lệ "cache miss"
5. Lưu `<post_id, user_id>` vào cache

#### Newsfeed retrieval deep dive

![Screen Shot 2022-09-14 at 23 23 31](https://user-images.githubusercontent.com/15076665/190181417-d333da01-4246-4bf3-bfb2-504eeeacbe8b.png)

Các static content như `images`, `videos` sẽ được lưu trữ ở CDN để tăng tốc độ truy vấn

Ở bước 4: news feed service sẽ lấy post ID list từ news feed cache
Ở bước 5: do một post không chỉ có đơn thuần ID mà còn có `user name`, `post content`, ... vì thế nên sẽ lấy các thông tin này ra từ `user cache` hoặc `user DB` cũng như `post cache` hoặc `post DB`

### Cache Architecture

Cache sẽ được chia thành 5 tầng như sau:

![Screen Shot 2022-09-15 at 8 28 26](https://user-images.githubusercontent.com/15076665/190280297-fac4ea18-1180-46f8-8c6b-0bbbf9ae0889.png)

- **News feed**: lưu trữ ID của news
- **Content**: lưu trữ post data, các post phổ biến sẽ được lưu trong `hot cache`
- **Social Graph**: lưu trữ dữ liệu về mối quan hệ giữa các user
- **Action**: lưu trữ thông tin về các hành động của user đối với post như `like`, `comment`, ...
- **Counters**: lưu counter cho likes, followers, ...

## Bước 4: Tổng kết

> Không có một thiết kế hoàn hảo nào cả. Tuỳ thuộc vào yêu cầu đặc trưng của từng hệ thống mà đưa ra thiết kế sao cho phù hợp nhất
