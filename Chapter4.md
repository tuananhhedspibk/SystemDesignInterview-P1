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

Các cloud microservice hiện nay đều tích hợp `rate limiting` bên trong `API gateway` và bây giờ chúng ta chỉ cần biết `API Gateway` sẽ được triển khai như một middleware có tích hợp `rate limiting`

Thế nhưng lúc này một câu hỏi được đặt ra đó là nên implement `rate limiting` tại `server side` hay `API Gateway`. Thực tế sẽ không có câu trả lời tuyệt đối cho vấn đề này vì nó phụ thuộc vào nhiều yếu tố như `tech stack` hoặc `engineer resource` và cả `thời gian`.
- Nếu triển khai bên phía server, bạn có thể kiểm soát hoàn toàn được `rate limiter` từ giải thuật cho đến quá trình vận hành
- Nếu bạn đang sử dụng cloud microservice với `API Gateway` có tích hợp `rate limiter` thì nên sử dụng luôn limiter của API gateway
- Việc triển khai rate limiter tốn tương đối thời gian nên nếu thời gian không cho phép thì lựa chọn `limiter của API gateway` hoặc `third-party` hoàn toàn có thể được cân nhắc

**Giải thuật dùng cho rate limiting**
Có khá nhiều giải thuật có thể cân nhắc như:
- Token bucket → đây là giải thuật hay được sử dụng nhất
- Leaking bucket
- Fixed window counter
- Sliding window log
- Sliding window counter

Giải thuật **token bucket** có thể được trình bày như sau:

![Screen Shot 2022-07-31 at 13 39 37](https://user-images.githubusercontent.com/15076665/182010410-72705e48-1db5-4808-bfab-3895ec8ea657.png)

![Screen Shot 2022-07-31 at 13 39 43](https://user-images.githubusercontent.com/15076665/182010413-5a5dac81-9d41-43b8-bac9-55a6c48fd21a.png)

Giải thuật **Leaking bucket** có thể được trình bày như sau:

![Screen Shot 2022-07-31 at 13 44 47](https://user-images.githubusercontent.com/15076665/182010518-7be43710-612c-418e-8699-e6359d8c163f.png)

**High-level architecture**
Ở level cao, chúng ta cần `counter` để có thể theo dõi số lượng requests gửi từ cùng một user hoặc địa chỉ IP nếu `counter` lớn hơn limit thì request sẽ bị drop.

Vấn đề ở đây là chúng ta sẽ lưu `counter` ở đâu. Nếu lưu trong DB thì sẽ tốn thời gian truy xuất, do đó lưu nó trong bộ nhớ (in-memory cache) là một sự lựa chọn hợp lí về:
- Thời gian truy xuất (nhanh)
- Hỗ trợ time-based expiration strategy

→ Và `Redis` chính là một sự lựa chọn hợp lí ở đây (bản thân nó là in-memory store với 2 câu lệnh INCR và EXPIRE)

High-level architecture của rate limit sẽ trông như sau:

![Screen Shot 2022-07-31 at 14 05 59](https://user-images.githubusercontent.com/15076665/182011034-852d4c9c-6e81-421a-889a-04e3291efb5e.png)
