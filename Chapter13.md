# Chương 13: Thiết kế search autocomplete system

Tính năng tự động hoàn thiện hoặc gợi ý còn gọi là `typeahead` hoặc `search-as-you-type` hoặc `incremental search`

Khi phỏng vấn, câu hỏi thiết kế hệ thống kiểu này còn có thể diễn đạt thành `design top k` hoặc `design top k most searched queries`

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Dưới đây là những tiêu chí cần làm rõ:

- Liệu hệ thống chỉ hỗ trợ matching ở phần đầu của query hay còn hỗ trợ phần giữa của query ? (Chỉ phần đầu)
- Có bao nhiêu gợi ý mà hệ thống sẽ trả về ? (5 gợi ý)
- Tiêu chí đưa ra gợi ý ? (Dựa theo độ phổ biến hoặc tần suất query trong lịch sử)
- Hệ thống có hỗ trợ spell check ? (Không hỗ trợ spell check hoặc autocorrect check)
- Search queries chỉ cho tiếng Anh ? (Chỉ cho tiếng Anh, nếu có thời gian thì nên hỗ trợ đa ngôn ngữ)
- Query có cho phép chữ hoa hay kí tự đặc biệt không ? (Không, chỉ cho phép chữ cái thường)
- DAU là 10 triệu

Tóm lược lại các yêu cầu chính:

- Fast response time: Khi user gõ search query thì ngay lập tức phải đưa ra gợi ý autocomplete cho người dùng
- Tính liên quan: các gợi ý nên liên quan đến search query
- Sắp xếp: kết quả trả về nên được sắp xếp theo mức độ phổ biến hoặc theo một ranking model nhất định
- Scalable: Hệ thống có thể xử lí high traffic volume
- High available: hệ thống phải được đảm bảo luôn sẵn sàng kể cả khi có một thành phần offline hoặc gặp sự cố mạng

### Envelope estimate

- DAU: 10 triệu
- 1 user search 10 lần / ngày
- 20 bytes data cho mỗi câu query
  - Giả sử dùng ASCII để mã hoá nên 1 char = 1 byte
  - Một câu query có trung bình 4 từ
  - 1 từ có trung bình 5 chars nên cả câu query = 5 * 4 = 20 bytes
- Với mỗi char người dùng nhập vào sẽ là 1 request lên server để lấy về suggestion, nên 1 câu query trung bình sẽ gửi 20 requests
- Ta sẽ có `10,000,000 * 10 * 20 / (24 * 3600)` ~ 24,000 QPS
- Peak QPS ~ `24,000 * 2` = 48,000
- Giả sử 20% query là mới, ta có: `10 triệu * 10 * 20 * 20%` ~ 0.4GB dữ liệu mới cần lưu trữ mỗi ngày

## Bước 2: High level design

Ở high-level chúng ta có thể chia hệ thống thành 2 phần:

- **Data gathering service**: tập hợp query của user và xử lí real-time. Tuy vậy, việc xử lí một lượng lớn dữ liệu theo hướng real-time không hẳn là một sự lựa chọn tốt
- **Query service**: với mỗi câu query hoặc prefix, trả ra 5 kết quả gợi nhớ với tần suất được truy xuất cao nhất

### Data gathering service

Ta sẽ sử dụng một bảng chứa `query` và `frequency` của query đó. Cứ mỗi khi query được người dùng gửi lên, thì nó sẽ được đưa vào bảng này và cập nhật frequency.

![Screen Shot 2022-09-27 at 8 36 43](https://user-images.githubusercontent.com/15076665/192399076-106f71d6-2921-454f-9c89-ba438b558544.png)

### Query service

Giả sử ta có bảng lưu query và frequency tương ứng như hình bên dưới

![Screen Shot 2022-09-27 at 8 39 19](https://user-images.githubusercontent.com/15076665/192399293-bfe25292-d0c9-4036-8514-3b6c2dd8ae5a.png)

Khi người dùng gõ `tw` thì sẽ hiển thị top 5 câu query dựa theo `frequency` ở bảng nêu trên. Để lấy ra 5 câu query, ta sẽ thực thi câu SQL sau:

```SQL
SELECT * FROM frequency_table
WHERE query LIKE `prefix%`
ORDER BY frequency DESC
LIMIT 5;
```

Giải pháp này phù hợp khi database nhỏ, với database có quy mô lớn thì sẽ diễn ra hiện tượng bottleneck. Giải pháp cho database lớn sẽ được xem xét ở phần `deep-dive design`

## Bước 3: Design deep dive

Trong phần này chúng ta sẽ xem xét những thành phần sau:

- Tries data structure
- Data gathering service
- Query service
- Scale the storage
- Tries operation

### Tries data structure

Đây là cấu trúc dữ liệu chuyên dùng để truy xuất string (còn gọi là prefix tree). Tries có một vài đặc điểm chính sau:

- Cấu trúc giống tree
- Root lưu empty string
- Mỗi node sẽ chứa prefix string hoặc query
- Ở đây ta không lưu empty query

![Screen Shot 2022-09-28 at 7 47 02](https://user-images.githubusercontent.com/15076665/192650939-4d385ec0-dd5d-4f26-851f-1fd5cacf8469.png)

Hình trên mô tả cấu trúc của một Tries lưu các query string "tree", "true", "try", "toy", "win", "wish"

Để hỗ trợ sắp xếp theo tần suất (frequency), tần suất cũng sẽ được lưu trong node như hình dưới đây

![Screen Shot 2022-09-28 at 7 48 59](https://user-images.githubusercontent.com/15076665/192651163-881bdbc4-7d71-4769-9e5c-2546fba2e4a2.png)

ví dụ như ở hình trên ta thấy "true" có frequency = 35.

Trước khi đi vào một ví dụ cụ thể hơn, ta cần nắm những khái niệm sau đây:

- p: độ dài của prefix
- n: tổng số nodes của tries
- c: số lượng con của một node

Các bước để tìm k search queries phổ biến

- B1: Tìm prefix, thao tác này có độ phức tạp `O(p)`
- B2: Duyệt cây con với gốc là prefix node để tìm các valid child nodes. Ở đây valid child nodes là các nodes chứa query string hoàn chỉnh, thao tác này có độ phức tạp `O(c)`
- B3: Sắp xếp các nodes con dựa theo `frequency` để lấy ra top k, thao tác này có độ phức tạp `O(clogc)`

Hình dưới đây mô tả ví dụ với `k = 2` và `prefix = tr`

![Screen Shot 2022-09-28 at 8 17 18](https://user-images.githubusercontent.com/15076665/192653972-642bba1e-b1f8-4184-9b10-ac5144daae24.png)

Tổng độ phức tạp cho thuật toán này là `O(p) + O(c) + O(clogc)`

Thuật toán trên vẫn còn quá chậm khi trong trường hợp tệ nhất, ta cần duyệt toàn bộ `tries`. Sau đây là 2 cách để tối ưu:

- Giới hạn độ dài của prefix
- Cache top search queries ở các nodes

#### Giới hạn độ dài của prefix

Người dùng hiếm khi search một query dài, nên chúng ta có thể giả định `p` là một `small integer number` - giả sử là `50`. Nếu ta giới hạn độ dài của prefix thì độ phức tạp trong tìm kiếm prefix sẽ từ `O(p)` thành `O(1)`

#### Cache top search queries ở các nodes

Để tránh tình trạng duyệt toàn bộ tries, ta sẽ tiến hành cache các query thường xuyên được tìm kiếm ở các nodes. Việc làm này sẽ tăng lượng bộ nhớ cần lưu trữ, tuy nhiên nó lại cải thiện thời gian tìm kiếm lên đáng kể. Dưới đây là một ví dụ

![Screen Shot 2022-09-28 at 8 25 14](https://user-images.githubusercontent.com/15076665/192654713-4cf0c2bb-537a-4464-b48f-dc16132c111f.png)

Ta thấy rằng lúc này việc tìm k top search queries sẽ có độ phức tạp thời gian là `O(1)`. Vậy nên tổng thời gian tìm kiếm cho cả thuật toán chỉ còn `O(1)`

### Data Gathering Service
