# Chuơng 6: Thiết kế key-value store

Trong chương này chúng ta sẽ cùng nhau thiết kết key-value store với khả năng:

- put(key, value) - thêm cặp key-value
- get(key) - lấy ra value tương ứng với key

## Hiểu rõ vấn đề và đưa ra phạm vi thiết kế

> Không có một thiết kế nào là hoàn hảo cả, mọi thứ đều phải dựa theo yêu cầu dù rằng phải đánh đổi về mặt bộ nhớ

Ta cần thiết kế key-value store đáp ứng được những nhu cầu như sau:

- Size của key-value pair < 10KB
- Có khả năng lưu trữ dữ liệu lớn
- High availability
- High scalability: có khả năng mở rộng thành một hệ thống lớn
- Automatic scaling: tuỳ theo traffic mà thêm / bớt đi servers một cách tự động
- Độ trễ thấp

## Single server key-value store

Việc thiết kế hệ thống trên một server duy nhất không quá khó. Ta có thể lưu nó vào bộ nhớ dưới dạng hash table. Tuy nhiên ta cũng có thể áp dụng 2 biện pháp tối ưu sau:

- Nén dữ liệu
- Lưu dữ liệu thường xuyên được truy cập trong bộ nhớ, còn lại lưu trong ổ đĩa

Dù vậy thì key-value store cũng sẽ chạm tới ngưỡng đầy khá nhanh.

## Distributed key-value store

Khi tiến hành thiết kế distributed system, việc hiểu được nguyên lý CAP là rất quan trọng (CAP: Consistency, Availability, Partition Tolerance)

### Nguyên lí CAP

CAP cho rằng một hệ thống phân tán không thể đáp ứng một cách đồng thời trên 2 điều kiện trong số 3 điều kiện là consistency (tính đồng nhất), availability (tính sẵn có), partition tolerance (khả năng chịu phân mảnh)

**Consistency**: mọi client sẽ truy cập được data như nhau bất kể nodes nào

**Availability**: mọi client đều nhận được response khi request kể cả khi có node down

**Partition Tolerance**: partition ở đây ám chỉ việc tương tác giữa 2 nodes bị đứt quãng. Điều kiện này đề cập đến việc hệ thống vẫn có thể vận hành kể cả khi gặp sự cố mạng

Nguyên lí CAP chỉ ra rằng một trong số 3 điều kiện trên phải chấp nhận hi sinh cho hai điều kiện còn lại

Các hệ thống key-value store phân tán hiện nay được chia thành 3 loại tương ứng với tổ hợp 2 trong 3 điều kiện mà hệ thống có thể đảm bảo: **CA**, **CP**, **AP**

> Trong một hệ thống phân tán, dữ liệu thường xuyên được sao lưu nhiều lần

Như hình bên dưới, dữ liệu sẽ được sao lưu ra 3 nodes: n1, n2, n3

![Screen Shot 2022-08-07 at 17 30 50](https://user-images.githubusercontent.com/15076665/183282336-d5b49004-afe3-443a-ab58-72f4faeebdc0.png)

## Hệ thống phân tán trong thực tế

Trong thực tế thì việc mạng gặp sự cố là điều khó tránh khỏi. Khi đó chúng ta phải chọn giữa **consitency** và **availability**. Như ví dụ bên dưới, n3 bị sập, n1 và n2 không thể giao tiếp với n3. Mọi dữ liệu ghi vào n1, n2 đều không thể đến được n3 và ngược lại.

![Screen Shot 2022-08-07 at 17 34 10](https://user-images.githubusercontent.com/15076665/183282425-cc50d33a-b360-4a96-b47c-823968c7931f.png)

Nếu ta chọn ưu tiên consistency (CP system), mọi thao tác ghi đến n1, n2 sẽ bị block. Khi đó hệ thống sẽ không hoạt động nữa. Lấy ví dụ cho các hệ thống ngân hàng (các hệ thốngn này luôn yêu cầu tính đồng nhất dữ liệu cao). Khi xảy ra sự cố, hệ thống sẽ trả về lỗi trước khi tính đồng nhất của dữ liệu được đối ứng.

Với các hệ thống ưu tiên availability, ta vẫn sẽ cho các thao tác ghi đến n1, n2 khi n3 được phục hồi ta sẽ tiến hành đồng bộ hoá dữ liệu cho n3.

> Việc chọn điều kiện đáp ứng ưu tiên là rất quan trọng với các hệ thống phân tán

## System Components

Các core components và kĩ thuật sẽ bàn tới ở đây là:

- Data partition
- Data replication
- Consistency
- Inconsistency resolution
- Hadling failure
- System architecture diagram
- Write path
- Read path

### Data partition

Với các hệ thống lớn, việc dồn dữ liệu vào một server là điều không thể. Do đó ta sẽ tiến hành chia nhỏ dữ liệu thành các partition và đưa vào nhiều servers khác nhau. Có 2 vấn đề cần giải quyết ở đây:

- Phân bổ dữ liệu như nhau ở các server (tránh tình trạng 1 server bị quá tải trong khi các servers còn lại bị bỏ không)
- Giảm thiểu việc di chú dữ liệu khi nodes được thêm / xoá đi

Việc áp dụng consitent hashing ở chương trước sẽ giúp giải quyết 2 vấn đề trên

- **Automatic scaling**: server có thể được tự động thêm / xoá tuỳ theo lưu lượng
- **Heterogeneity**: số lượng nodes sẽ tỉ lệ thuận với dung lượng của server

### Data replication

Để đảm bảo high availability & reliability ta cần sao lưu dữ liệu (asynchronous replicated data) tới N servers (N có giá trị được thiết lập từ trước), ví dụ với N = 3, khi có key mới thêm vào ta sẽ đi theo chiều kim đồng hồ và sao lưu key vào 3 servers kề nhau liên tiếp

![Screen Shot 2022-08-07 at 18 05 57](https://user-images.githubusercontent.com/15076665/183283623-aaf7acd4-4b7f-4ea3-84e8-6160af294223.png)

Với virtual node, khi quét N virtual node thì số lượng server quét được có thể sẽ ít hơn N, do đó ta sẽ chỉ thực hiện quét trên server khi tiến hành đi theo chiều kim đồng hồ.

### Consitency

Dưới đây là một vài tham số cần lưu ý:

N: số lượng replicas
W: "túc số" của thao tác ghi. Một thao tác ghi được coi là thành công nếu theo tác này được công nhận từ W replicas
R: "túc số" của thao tác đọc. Một thao tác đọc chỉ được coi là thành công khi nó phải có khả năng đợi response từ ít nhất R replicas.

![Screen Shot 2022-08-07 at 18 21 14](https://user-images.githubusercontent.com/15076665/183284124-8186a905-6078-4960-ab42-b30dd9b12e0f.png)

Như hình trên W = 1, không có nghĩa là thao tác ghi chỉ được gửi đến 1 server. Thao tác này được coi là thành công nếu như coordinator nhận được ít nhất một ACK từ node (ví dụ s1), thì việc 2 nodes s0 và s2 có gửi ACK cho coordinator hay không sẽ không còn quan trọng nữa.

Ghi chú: coordinator hoạt động như một **proxy** giữa client với các nodes.

Việc configure N, W, R cần đảm bảo sự cân bằng nhất định giữa **latency** và **consistency**

Khi `W, R = 1` hệ thống sẽ trả về kết quả cho client nhanh hơn ghi W hoặc R > 1 vì khi đó coordinator sẽ cần phải chờ ACK từ những nodes chậm nhất. Tuy nhiên "nhờ" sự chậm chễ này mà tính đồng nhất **consistency** được đảm bảo.

Khi `W + R > N` thì tính đồng nhất sẽ rất cao do sẽ có ít nhất 1 nodes đảm bảo được việc:

- Chứa dữ liệu ghi
- Chứa dữ liệu đọc
mới nhất.

Làm thế nào để configure N, W, R phù hợp cho use-case của chúng ta ? Dưới đây là một vài sự lựa chọn:

- R = 1, W = N: fast read
- W = 1, R = N: fast write
- R + W > N: strong consitency
- R + W <= N: không đảm bảo strong consistency
