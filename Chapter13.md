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
