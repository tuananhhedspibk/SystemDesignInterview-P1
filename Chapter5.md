# Chuơng 5: Design consistent hashing

Khi tiến hành horizontal scaling, yếu tố ta cần phải đảm bảo đó là việc req/ data được phân bổ đồng đều và hiệu quả trên các servers.

`Consistent Hashing` là một kĩ thuật phổ biến để giải quyết bài toán trên.

Công thức chung thường dùng khi lưu key vào các cache servers đó là

```
serverIndex = hash(key) % N
```

Với N là số lượng server hoặc `server pool size`. Ta có thể thấy rõ hơn thông qua ví dụ trực quan sau:

<img width="867" alt="Screen Shot 2022-08-04 at 22 44 46" src="https://user-images.githubusercontent.com/15076665/182862715-5d2cb1a5-9636-4461-bf08-c90f1a774d5f.png">

Tuy nhiên cách làm này chỉ phát huy hiệu quả khi số lượng server là cố định, trong trường hợp `thêm server mới` hoặc `cắt giảm server` thì kết quả sẽ có sự khác biệt trông thấy dù vẫn dùng `hash function cũ`

<img width="867" alt="Screen Shot 2022-08-04 at 22 47 45" src="https://user-images.githubusercontent.com/15076665/182863183-b4181ba7-5b43-4a27-b0be-ce1097214bad.png">

Lúc này các key sẽ được phân bổ lại, dẫn đến tình trạng cache missed sẽ xảy ra rất thường xuyên.

## Consitent Hashing

Với giải thuật consistent hashing, trong trường hợp hash table được resize thì chỉ có `k/n` keys được phân bổ lại với (k: số lượng keys, n: số lượng server hoặc slot)

### Hash space, hash ring

Ta giả thiết sử dụng hàm `SHA-1` để tiến hành hashing. Với hàm `SHA-1` ta sẽ có cả thảy `2^160` giá trị từ `0 → 2^160 - 1`, gọi giá trị đầu ra lần lượt là `x0, x1, ..., xn`

Thay vì sử dụng một "mảng thẳng" ta sẽ sử dụng "mảng vòng" như sau:

<img width="930" alt="Screen Shot 2022-08-04 at 22 58 01" src="https://user-images.githubusercontent.com/15076665/182865654-48676c84-e619-4f3b-a59b-028d35a1f659.png">

### Hash servers

Ta phân bổ server trên hash ring như sau:

<img width="882" alt="Screen Shot 2022-08-04 at 22 59 52" src="https://user-images.githubusercontent.com/15076665/182865892-31641e95-1749-4c57-b829-214b64508e5b.png">
