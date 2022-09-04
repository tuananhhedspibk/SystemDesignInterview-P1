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

#### Consitency models

Tính nhất quán là một trong những yếu tố quan trọng cần cân nhắc khi thiết kế key-value store. Có 3 loại chính như sau:

- **Strong consitency**: đây là kiểu có tính nhất quán cao, mọi thao tác đọc đều được trả về dữ liệu mới nhất
- **Weak consistency**: dữ liệu trả về có thể không phải là dữ liệu mới nhất
- **Eventual consistency**: đây là một dạng đặc biệt của weak consistency, ở dạng này thì sau những khoảng thời gian nhất định, việc cập nhật dữ liệu sẽ được diễn ra trên mọi replicas.

**Strong consitency** - mọi thao tác đọc/ghi dữ liệu sẽ bị block cho đến khi mọi replica khác đồng ý. Với các hệ thống highly available thì model này không hề thích hợp

### Inconsistency resolution: versioning

![File_000](https://user-images.githubusercontent.com/15076665/188303602-d77559bc-f74a-4176-b001-a8f167277bca.png)

Như ở ví dụ trên ta có 2 versions cho cùng một biến với 2 giá trị khác nhau là "JohnNewYork" và "JohnCali"

Vấn đề ở đây sẽ nằm ở chỗ:

- Tìm ra conflict về mặt dữ liệu
- Giải quyết conflict

**Vector clock** là một giải pháp rất phổ biến ở đây.

Với mỗi data item ta sẽ có một cặp `[Si, vi]` với:

- Si: server thứ i lưu trữ data item
- vi: version thứ i

Như vậy một data item D sẽ có cấu trúc vector clock như sau `([S1, 1], [S2, 1], [S1, 2], ...)`. Mô tả chi tiết về vector clock sẽ như sau:

![Screen Shot 2022-09-04 at 17 02 29](https://user-images.githubusercontent.com/15076665/188303704-0be73f29-8680-4998-80a4-91d6efcb76c8.png)

Chú ý: Khi mới được tạo thì vector clock pair sẽ có dạng `[Si, 1]`. Cứ sau mỗi lần ghi, thì vi sẽ tự động tăng 1.

Ở đây ta nói version X là "tổ tiên" của version Y nếu mọi vector clock trong X đều nhỏ hơn hoặc bằng Y (VD: D([S0, 1], [S1, 1]) là tổ tiên của D([S0, 1], [S1, 2]))

Tương tự, ta nói rằng version X là "ngang bằng" với version Y nếu tồn tại trong cả 2 version những vector clock lớn và nhơ hơn nhau. Ví dụ: `Version X = D([S0, 1], [S1, 2])` và `Version Y = D([S0, 0], [S1, 3])

Dù rằng vector clock giúp ta giải quyết conflict thế nhưng nó yêu cầu client cần triển khai logic xử lí conflict. Hơn nữa, số lượng `[server: version]` pair sẽ tăng rất nhanh. Lúc đó sẽ phải xoá đi các "tổ tiên" nếu số lượng pair này vượt quá một ngưỡng cho phép.

Khi đó sẽ phát sinh thêm vấn đề về tìm "tổ tiên xa xưa" cho các pair.

### Handling Failure

#### Failure detection

Trong một hệ thống phân tán, ta thường dựa vào việc ít nhất có 2 server báo rằng 1 server khác bị "sập" để đưa ra kết luận rằng server "sập" đó thật sự "sập".

Trên thực tế, cơ chế all-to-all multicasting thường dùng để liên lạc giữa các servers như hình bên dưới. Tuy nhiên cơ chế này sẽ không hiệu quả khi số lượng server quá lớn.

![Screen Shot 2022-09-04 at 17 24 36](https://user-images.githubusercontent.com/15076665/188304561-2c3c9296-edd7-43de-b34a-9ade298eafff.png)

Có một cách khác để tìm ra server "sập" đó là **gossip protocol**. Cụ thể như sau:

- Mỗi node sẽ lưu một `member_ship_list`
- Mỗi node và mỗi member trong list ở trên sẽ có `memberId`, `heart_beat_counter`
- Node sẽ đều đặn tăng `heart_beat_counter` của mình lên
- Node sẽ gửi `heart_beat_counter` cho các nodes ngẫu nhiên, các nodes ngẫu nhiên này cũng sẽ gửi tiếp cho các nodes khác
- Khi nhận được `heart_beat_counter` thì node sẽ cập nhật thông tin của `member_ship_list`
- Nếu sau một khoảng thời gian nhất định mà `heart_beat_counter` không tăng thì có thể kết luận member (server) này bị "sập"

![File_000 (1)](https://user-images.githubusercontent.com/15076665/188304895-7fbf85d5-d184-4fc0-87b2-e6db87cc6b52.png)

![Screen Shot 2022-09-04 at 17 39 01](https://user-images.githubusercontent.com/15076665/188305062-a29521d9-0ae3-4e02-868a-f8e790ea6c75.png)

#### Handling temporary failure

Sau khi phát hiện lỗi, ta cần có một cơ chế để xử lí lỗi. Sử dụng quorum cũng là một cách tiếp cận nhưng sẽ phát sinh một vấn đề đó là các thao tác "read" và "write" có thể bị block nếu điều kiện về quorum không được đảm bảo.

Một cách tiếp cận khác được gọi là `sloppy quorum` có thể được sử dụng. Thay vì đảm bảo yêu cầu về quorum, hệ thống sẽ:

- Chọn ra `W servers khoẻ mạnh` đầu tiên để phục vụ cho thao tác "write"
- Chọn ra `R servers khoẻ mạnh` đầu tiên cho thao tác "read"

Nếu một server bị "sập", hệ thống sẽ chọn ra một server khác thay thế nó tạm thời cho đến khi server "sập" kia được phục hồi. Quá trình này được gọi là `hinted handoff`

![Screen Shot 2022-09-04 at 18 23 07](https://user-images.githubusercontent.com/15076665/188306625-784a1d99-668b-4bcf-a3a3-fbcc018a6af4.png)

#### Handling permanent failure

`Hinted hanoff` chỉ xử lí được khi server sập "tạm thời", còn nếu sập "vĩnh viễn thì sao". Với tình huống như vậy ta sẽ triển khai `anti-entropy protocal` để giữ các replicas được đồng bộ.

`Anti-entropy protocol` đề xuất việc so sánh từng phần để tìm ra những phần khác biệt giữa các replicas. Ở đây ta sẽ sử dụng `Merkle tree` (các nodes không phải là node lá sẽ được gán nhãn dựa theo phép băm các nhãn của các nodes con)

Giả sử key space của ta đi từ 1 → 12

B1: Chia các keys theo bucket (1 bucket sẽ có 3 keys)

![Screen Shot 2022-09-04 at 18 33 38](https://user-images.githubusercontent.com/15076665/188307077-5af0fc40-cf06-4b2c-99ba-018e366b6c31.png)

B2: Với từng buckets ta sẽ băm từng key bằng một hàm hash dùng chung cho mọi replicas

![Screen Shot 2022-09-04 at 18 34 17](https://user-images.githubusercontent.com/15076665/188307079-a502d633-cb09-4bb3-9ae8-307f9b58117a.png)

B3: Tạo một hash node cho từng bucket

![Screen Shot 2022-09-04 at 18 35 30](https://user-images.githubusercontent.com/15076665/188307140-96b3cc5c-892d-4a8a-8b41-5355b56ce07a.png)

B4: Build tree cho đến khi chạm tới root node

![Screen Shot 2022-09-04 at 18 35 52](https://user-images.githubusercontent.com/15076665/188307142-cd4a8733-f565-4cbd-bc65-658bef1f660c.png)

B5: So sánh root node, nếu bằng nhau thì OK, nếu không lần lượt so sánh các cây con trái và phải cho đến khi tìm ra bucket khác nhau và tiến hành đồng bộ hoá

#### Handling data center outage

Phần này nói về việc xử lí khi một data center ngưng hoạt động. Trên thực tế dữ liệu sẽ được sao lưu ra nhiều data centers khác nhau và nếu một data center gặp sự cố thì request sẽ được chuyển tiếp đến data center khác.

**System architecture diagram**
![Screen Shot 2022-09-04 at 18 46 39](https://user-images.githubusercontent.com/15076665/188307568-0452171e-f515-489b-8bdf-91768e6c5ef8.png)

Với kiến trúc trên:

- Client sẽ tương tác với key-value store thông qua các API `get`, `put`
- Coordinator node sẽ hoạt động như một proxy giữa client và key-value store
- Hệ thống sẽ được phân tán hoàn toàn
- Không có tình trạng `Single Point Of Failure` khi mọi nodes đều có nhiệm vụ như nhau
- Dữ liệu sẽ được sao lưu trên mọi nodes
- Các nodes được phân tán dựa trên giải thuật `consitent hashing`

Mỗi một node sẽ được trang bị đảm nhiệm những chức năng như sau:

- Client API
- Conflict resolution
- Failure detection
- Storage Engine
- Replication

**Write Path**
![Screen Shot 2022-09-04 at 18 54 29](https://user-images.githubusercontent.com/15076665/188307835-211d9fe7-8e62-4939-9dbc-7074207d6005.png)

① Write operation sẽ được lưu vào log
② Dữ liệu mới sẽ được lưu vào memory cache
③ Khi memory cache đầy hoặc chạm một ngưỡng nào đó, mọi sự thay đổi sẽ được đưa vào SSTable (Sorted String table)

**Read Path**
![Screen Shot 2022-09-04 at 18 59 19](https://user-images.githubusercontent.com/15076665/188307973-41d5f6ae-a5a1-49a4-b4da-c7bcfd87cbf1.png)

Nếu cache hit, thì data sẽ được trả về client như hình minh hoạ ở trên. Trường hợp ngược lại

![Screen Shot 2022-09-04 at 19 00 33](https://user-images.githubusercontent.com/15076665/188308014-66a20ae9-794c-4b61-9fef-702131dabb7f.png)

② Check bloom filter
③ Bloom filter sẽ tìm ra SSTable nào chứ key
④ SSTable tìm và trả ra dữ liệu
⑤ Trả về dữ liệu cho client
