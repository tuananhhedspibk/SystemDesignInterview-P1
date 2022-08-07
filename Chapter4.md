# Chương 4: Thiết kế rate limiter

Trong network system, rate limiter dùng để điều khiển traffic rate gửi từ phía client. Với HTTP thì rate limiter sẽ giới hạn số request gửi từ phía client trong một khoảng thời gian nhất định. Nếu số lượng request vượt quá giới hạn trên thì mọi lời gọi đến hệ thống đều bị từ chối.

Trong thực tế rate limiter sẽ giúp

- Phòng tránh DoS attack.
- Giảm chi phí vận hành khi giảm được số lượng server xử lí request
- Giảm tải cho server

## B1: Hiểu vấn đề và thiết lập phạm vi thiết kế

Có khá nhiều loại giải thuật cho rate limiting, việc trao đổi với interviewer sẽ giúp bạn phần nào hình dung được loại rate limiter cần phải build.

## B2: Đưa ra high-level design

**Nên đặt rate-limiter ở đâu**
Có 2 nơi để đặt đó là **client side** và **server side**.

- **Client Side**: đây là nơi không đáng tin cậy để đặt rate-limiter vì có thể bị ảnh hưởng bởi những `hành động đáng nghi` của client.

- **Server side**: như hình dưới đây

![img](https://user-images.githubusercontent.com/15076665/182004864-a881aa70-ac4d-4487-a7e7-af60055cc116.png)

Hoặc cũng có thể đặt nó ở vị trí middleware như dưới đây

![img](https://user-images.githubusercontent.com/15076665/182004865-19f56914-bb1c-45ee-99ab-ddfa114a0626.png)

Với cách tiếp cận thông qua middleware như trên ta có thể mô tả sơ bộ cách hoạt động của rate limiter như sau:

![img](https://user-images.githubusercontent.com/15076665/182004901-25e3319b-efde-49d7-a3d9-66588be1489d.png)

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

Client gửi request đến rate limiting middleware, middleware sẽ lấy giá trị counter từ phía redis về. Nếu trong trường hợp:

- Đạt tới giá trị limit thì request sẽ bị rejected
- Chưa đạt tới giá trị limit thì req sẽ được chuyển tới API và counter tăng 1

## B3: Thiết kế chi tiết

**Rate limiting rules** sẽ được lưu trữ tại các config files trên đĩa

**Xử lí khi vượt quá rate limit**: thông thường ta sẽ trả về lỗi 429 (too many requests) cho phía client, nhưng trong một số trường hợp ta có thể lưu các req vượt quá limit lại để xử lí sau đó.

**Rate limiter headers**: các thông tin như client có đang bị "thắt cổ chai" hay không hoặc số lượng req còn lại mà client có thể gửi trước khi bị "thắt cổ chai" sẽ được đưa vào `HTTP response header` dưới các thuộc tính sau:

- X-Ratelimit-Remaining
- X-Ratelimit-Limit
- X-Ratelimit-Retry-After

Khi user gửi quá nhiều req thì:

- 429 Error Code sẽ được trả về
- X-Ratelimit-Retry-After

Detail Design sẽ như sau:

![Screen Shot 2022-07-31 at 16 37 05](https://user-images.githubusercontent.com/15076665/182015305-59cac443-730b-4722-83b2-542b132cd6d8.png)

Trong hình trên các rules sẽ được lưu trữ trên đĩa và sẽ được các workers kéo và lưu trữ trên cache khi sử dụng

**Rate limiter trong môi trường phân tán**
Khi xây dựng rate limiter trên nhiều servers sẽ phát sinh hai vấn đề tiêu biểu sau:

1. Race condition:
Có thể hiểu như việc cùng dùng chung một tài nguyên là counter được lưu trong redis sẽ dẫn đến tình trạng không toàn vẹn về dữ liệu.

![Screen Shot 2022-07-31 at 16 46 38](https://user-images.githubusercontent.com/15076665/182015620-4ec9ac51-7c5a-4dfa-8e52-7c4fdef941ff.png)

Với vấn đề này thì `lock` là biện pháp xử lí phổ biến nhất, thế nhưng `lock` sẽ làm chậm đi tốc độ của hệ thống do đó ta có thể sử dụng `Lua script` hoặc `Sorted set của Redis`

- Synchornization issue
Nếu có nhiều user thì một rate limiter là không đủ và do đó ta cần nhiều rate limiter một lúc. Khi sử dụng nhiều limiter cùng một lúc đòi hỏi ta phải đồng bộ hoá dữ liệu giữa chúng, như ví dụ dưới đây do web tier là stateless thì các client có thể gửi req đến bất kì rate limiter nào.

Nếu không có sự đồng bộ giữa các rate limiter thì ví dụ `rate limiter 1` sẽ không thể biết bất kì thông tin nào về `client 2` dẫn đến việc hoạt động của rate limiter không được như ý muốn

![Screen Shot 2022-07-31 at 16 59 29](https://user-images.githubusercontent.com/15076665/182016000-f6f06fb5-4173-4e94-a532-95bd21255118.png)

Một giải pháp có thể áp dụng đó là sử dụng `sticky session` để cho phép một client có thể gửi req đến cùng một rate limiter → Cách làm này khó có thể mở rộng trong tương lai.

Có một cách khác đó là sử dụng `centralized data stores như Redis`.

![Screen Shot 2022-07-31 at 17 03 55](https://user-images.githubusercontent.com/15076665/182016151-b1d9887a-c4ad-4f5c-bdbd-a60243af53c6.png)

### Performance improvement

1. Việc setup các data center sẽ ảnh hưởng đến rate limiter vì độ trễ với các user ở xa data center sẽ lớn.
2. Đồng bộ hoá dữ liệu với eventual consistency model

### Monitoring

Cần phải phân tích dữ liệu để biết rate limiter nào đang hoạt động hiệu quả, về cơ bản ta muốn:

- Rate limiting algorithm hoạt động hiệu quả
- Rate liminting rules nào hoạt động hiệu quả

Ví dụ:

- Nếu rate limiting rule quá nghiêm ngặt dẫn đến nhiều valid request bị bỏ thì ta có thể điều chỉnh rule đi đôi chút.
- Hoặc nếu rate limiter không hoạt động, dẫn tới tình trạng traffic tới server tăng đột biết thì ta cần thay thế rate limiting algorithm để tránh tình trạng "bùng nổ" traffic như trên

**B4: Tổng kết**
Bạn có thể bàn luận thêm với interviewer về một vài vấn đề như:

- Hard vs soft rate limiting: **Hard** nghĩa là số lượng req phải nhỏ hơn rate limiting, **Soft** số lượng req có thể vượt qua giới hạn trong một khoảng thời gian nhất định
- Triển khai ở layer khác, ở ví dụ trên ta chỉ triển khai ở `Application Layer: layer 7` còn nếu thiết lập rate limiter dựa theo users IP thì ta có thể triển khai ở `Network layer: layer 3`

Ngoài ra ta cũng có thể tránh `rate limited` ở phía client bằng cách:

- Sử dụng client cache để tránh việc gửi req lên API server thường xuyên
- Xử lí các exception, error để client có thể phục hồi từ exception
- Set back off time cho retry logic
