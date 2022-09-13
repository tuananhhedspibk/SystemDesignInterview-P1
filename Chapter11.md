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

- Feed publishing API

*POST /v1/me/feed*
params:
  ① content: nội dung của post
  ② auth_token: dùng để xác thực API req

- Newsfeed retrieval API

*GET /v1/me/feed*
params:
  ① auth_token: dùng để xác thực API req

![Screen Shot 2022-09-14 at 8 25 34](https://user-images.githubusercontent.com/15076665/190026722-ae5e7d2b-0f54-4657-abb2-cb595e8314af.png)
