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

### Hash keys

Có một điều khác biệt ở đây đó là phép băm sẽ không đi kèm với phép module. Các keys sẽ được băm và phân bổ như sau:

<img width="860" alt="Screen Shot 2022-08-06 at 22 40 10" src="https://user-images.githubusercontent.com/15076665/183251559-de4af599-2a85-4e16-a860-9faf6086a8ef.png">

### Server lookup

Để tìm server lưu trữ key ta sẽ đi theo chiều kim đồng hồ cho đến khi tìm được server thì thôi.

### Add server

Giả sử ta thêm server 4 như hình vẽ bên dưới, lúc này chỉ có key0 là phải redistributed các keys còn lại không cần phải redistributed.

<img width="899" alt="Screen Shot 2022-08-06 at 22 51 16" src="https://user-images.githubusercontent.com/15076665/183251768-90449843-a72d-415c-95ee-56d5de10f982.png">

### Remove server

Khi một server bị loại bỏ thì chỉ có một số lượng nhỏ các keys được re-distributed.

<img width="758" alt="Screen Shot 2022-08-06 at 22 53 14" src="https://user-images.githubusercontent.com/15076665/183251844-81881b57-6219-4370-8483-2fc871fd2280.png">

### Hai vấn đề cơ bản của cách tiếp cận

Cách tiếp cận của thuật toán consistent hashing như sau:
1. Map servers và keys sử dụng cùng một hàm hash như nhau
2. Để tìm ra server lưu trữ key ta chỉ cần đi theo chiều kim đồng hồ trên hash ring cho đến khi gặp được server đầu tiên

Có thể thấy rõ 2 vấn đề của cách tiếp cận này đó là:
- Sẽ rất khó để giữ cho partition size giữa các server đều nhau khi có một hoặc nhiều servers được thêm mới hoặc bị loại bỏ đi (partition size là kích cỡ / độ rộng của khoảng cách giữa 2 servers liên tiếp)
- Trên thực tế partition size giữa các server hoặc sẽ rất nhỏ hoặc sẽ rất lớn như hình minh hoạ dưới đây

<img width="929" alt="Screen Shot 2022-08-06 at 22 58 23" src="https://user-images.githubusercontent.com/15076665/183252046-9cba50c6-2109-4cee-8a39-0658ba9033b2.png">

Và trên thực tế thì sự phân bổ keys này cũng không đồng đều

Kĩ thuật **Virtual node** như sau sẽ được dùng để giải quyết vấn đề trên.
Virtual node ở đây sẽ được dùng để biểu diễn các servers. Ví dụ với một server ta sẽ biểu diễn nó bằng 3 virtual nodes chẳng hạn (trên thực tế con số này lớn hơn rất nhiều). Như hình dưới đây

<img width="912" alt="Screen Shot 2022-08-06 at 23 04 36" src="https://user-images.githubusercontent.com/15076665/183252285-e420c442-840d-4f12-bd00-99ddc746b0b1.png">

Server s0 sẽ được biểu diễn bằng 3 nodes: s0_0, s0_1, s0_2. Các cạnh biểu diễn bằng s0 sẽ do server s0 quản lí.

Để tìm ra server lưu trữ key, ta sẽ đi theo chiều kim đồng hồ, gặp virtual node biểu diễn server nào thì key sẽ thuộc server đó.

<img width="860" alt="Screen Shot 2022-08-06 at 23 09 18" src="https://user-images.githubusercontent.com/15076665/183252448-ed1aafdb-6b1c-4d61-8880-076d70202c01.png">

Khi số lượng server / node tăng thì đồng nghĩa với việc hash ring sẽ trở nên "cân bằng" hơn.

Một nghiên cứu cho thấy rằng với khoảng 100 nodes ta sẽ có độ lệch tiêu chuẩn (chỉ số đo độ dàn trải của dữ liệu) là khoảng 10%, 200 nodes sẽ là 5%. Tuy nhiên tuỳ theo yêu cầu của hệ thống ta có thể điều chỉnh số lượng nodes cho phù hợp vì bản thân ta vẫn cần nhiều không gian để lưu trữ keys.

### Cách tìm các keys bị ảnh hưởng

Khi một server được thêm mới, ta sẽ lấy toàn bộ keys nằm giữa server mới thêm và server kề trước nó ngược chiều kim đồng hồ, như ví dụ dưới đây toàn bộ keys nằm giữa server 4 (mới thêm) và server 3 (kề trước server 4) sẽ bị ảnh hưởng.

<img width="904" alt="Screen Shot 2022-08-06 at 23 16 40" src="https://user-images.githubusercontent.com/15076665/183252753-c01210f8-0e0b-481b-ab17-b0ed2b1d3b48.png">

Cũng tương tự như khi server bị loại bỏ

<img width="861" alt="Screen Shot 2022-08-06 at 23 18 45" src="https://user-images.githubusercontent.com/15076665/183252797-5370bd33-acd2-4adc-a32a-daae9ddc1dec.png">

### Wrap up

Sau chương này ta có thể thấy được một vài những ưu điểm của giải thuật consitent hashing như sau:
- Giảm thiểu số lượng keys phải re-distributed khi có server được thêm mới hoặc bị loại bỏ
- Tăng khả năng horizontal scaling vì dữ liệu được phân bố tương đối đồng đều
- Giảm thiểu hotspot problem. Đây là vấn đề khi một shard bị truy cập quá tải, consistent hashing giúp phân bổ tương đối đồng đều các keys nên có thể giảm thiểu hotspot problem.
