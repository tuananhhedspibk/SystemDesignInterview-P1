# Chương 4: Thiết kế rate limiter

Trong network system, rate limiter dùng để điều khiển traffic rate gửi từ phía client. Với HTTP thì rate limiter sẽ giới hạn số request gửi từ phía client trong một khoảng thời gian nhất định. Nếu số lượng request vượt quá giới hạn trên thì mọi lời gọi đến hệ thống đều bị từ chối.

Trong thực tế rate limiter sẽ giúp
- Phòng tránh DoS attack.
- Giảm chi phí vận hành khi giảm được số lượng server xử lí request
- Giảm tải cho server

**B1: Hiểu vấn đề và thiết lập phạm vi thiết kế**

Có khá nhiều loại giải thuật cho rate limiting, việc trao đổi với interviewer sẽ giúp bạn phần nào hình dung được loại rate limiter cần phải build.

**B2: Đưa ra high-level design**

**Nên đặt rate-limiter ở đâu**
Có 2 nơi để đặt đó là **client side** và **server side**.

- **Client Side**: đây là nơi không đáng tin cậy để đặt rate-limiter vì có thể bị ảnh hưởng bởi những `hành động đáng nghi` của client.

- **Server side**: như hình dưới đây

<img width="860" alt="Screen Shot 2022-07-31 at 9 39 59" src="https://user-images.githubusercontent.com/15076665/182004864-a881aa70-ac4d-4487-a7e7-af60055cc116.png">

Hoặc cũng có thể đặt nó ở vị trí middleware như dưới đây

<img width="891" alt="Screen Shot 2022-07-31 at 9 40 07" src="https://user-images.githubusercontent.com/15076665/182004865-19f56914-bb1c-45ee-99ab-ddfa114a0626.png">

Với cách tiếp cận thông qua middleware như trên ta có thể mô tả sơ bộ cách hoạt động của rate limiter như sau:

<img width="906" alt="Screen Shot 2022-07-31 at 9 41 49" src="https://user-images.githubusercontent.com/15076665/182004901-25e3319b-efde-49d7-a3d9-66588be1489d.png">

Giả sử hệ thống chỉ cho phép client gửi tối đa 2req/s nhưng client lại gửi những 3req/s thì 2req đầu tiên sẽ được truyền tới phía server còn req thứ 3 sẽ bị reject và trả lại `HTTP status code 429` cho phía client.
