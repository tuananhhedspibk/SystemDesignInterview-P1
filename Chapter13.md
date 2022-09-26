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

