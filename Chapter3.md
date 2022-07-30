# Chương 3: Framework cho system design interviews

System design interview sẽ mô phỏng lại quá trình giải quyết một vấn đề trong thực tế mà ở đó bạn phải hợp tác với người khác để cùng nhau đưa ra hướng giải quyết cho một vấn đề vẫn còn mơ hồ.

> Kết quả cuối cùng không quan trọng bằng quá trình bạn làm việc để đưa ra design

Điều mà người phỏng vấn mong chờ ở những ứng viên của mình đó là:
- Khả năng hợp tác làm việc
- Khả năng làm việc dưới áp lực
- Khả năng giải quyết những vấn đề vẫn còn mơ hồ

Trên thực tế thì **technical skills chỉ là một phần mà thôi**

> Một kĩ năng quan trọng khác đó là hỏi những câu hỏi thông minh

Một trong những "vấn nạn" của những kĩ sư IT đó là việc **over-engineering**, khi họ quá tập trung vào khía cạnh kĩ thuật mà không cân nhắc đến các vấn đề khác như:
- Trade-off
- Chi phí vận hành

Dưới đây là 4 bước tiêu biểu của một quá trình system design interview
**B1: Hiểu vấn đề và đưa ra design scope** - 3~10 phút

Trong system design interview, việc đưa ra câu trả lời quá nhanh mà thiếu đi sự cân nhắc hay suy nghĩ là một điểm trừ đáng kể.

> Không nên đưa ra câu trả lời ngay, hãy ngẫm nghĩ, hỏi những câu hỏi cần thiết về requirement

Khi chủ động hỏi thì interviewer sẽ đưa ra câu trả lời hoặc đưa ra cho bạn những gợi ý. Với những gợi ý **hãy viết những suy nghĩ hay giả định của bạn ra giấy** vì có thể sau đó bạn sẽ cần đến nó

Dưới đây là những câu hỏi bạn nên hỏi:
- Chúng ta sẽ build tính năng gì ?
- Có bao nhiêu người dùng ?
- Có ý định scale-up như thế nào ? Sau 3 tháng, 6 tháng, 1 năm thì muốn lần lượt scale phần nào ?
- Technology stack ?
- Có muốn tận dụng service có sẵn để đơn giản hoá quá trình design hay không ?

**B2: Đề xuất high-level design** 10~15 phút
Ở bước này chúng ta sẽ đưa ra high-level design và cố gắng nhận được sự đồng thuận từ phía interviewer. Quá trình này sẽ gồm những bước con như sau:
- Đưa ra bản nháp sơ bộ về design, trao đổi với interviewer giống một teammate
- Phác thảo các main components (có thể là client, API, cache, DB, ...)
- Thực hiện back-of-the-envelope calculation để nhận định xem bản nháp design có khớp với yêu cầu về scaling hay không.

Với ví dụ về thiết kế một hệ thống publish news và feed news ta có thể chia thành 2 luồng chính như sau:
- Publish news

![Screen Shot 2022-07-30 at 18 40 04](https://user-images.githubusercontent.com/15076665/181904774-a0bd8ea7-8ebe-4675-96f2-8acb16d7461b.png)

- Feed news

![Screen Shot 2022-07-30 at 18 40 10](https://user-images.githubusercontent.com/15076665/181904777-16945512-994f-426b-b7a1-6780ceaec99d.png)

**B3: Design Deep Dive** 10~25 phút
Ở bước này bạn và interviewer nên hoàn tất những mục tiêu như sau:
- Đạt thoả thuận về mục tiêu tổng quan và feature scope
- Phác thảo thiết kế tổng thể (high-level overall design)
- Nắm được phản hồi từ phía interviewer về high-level design
- Nhận biết được nên tập trung vào phần nào dựa theo phản hồi từ phía interviewer

Trên thực tế thì interviewer sẽ yêu cầu bạn tập trung vào một vài components cụ thể

Với ví dụ về thiết kế một hệ thống publish news và feed news ta sẽ đưa ra `detail design` cho 2 chức năng này như sau:

- Publish news

![Screen Shot 2022-07-30 at 18 51 17](https://user-images.githubusercontent.com/15076665/181905096-4d8e9caa-fae1-440f-a1fa-a4299ff2025d.png)

- Feed news

![Screen Shot 2022-07-30 at 18 51 37](https://user-images.githubusercontent.com/15076665/181905099-45bc7f58-285f-460d-86a4-6256c711b51e.png)

**B4: Tổng kết** 3~5 phút
Phần này sẽ là phần hỏi thêm các câu hỏi phụ khác hoặc thảo luận thêm ví dụ như:
- Về bottlenecks, improvement → critical thinking (tự nhận định bản thân)
- Error cases
- Monitor metrics, error logs
- Next scale (hiện hệ thống chỉ thiết kế cho 1 triệu người nhưng với 10 triệu người thì sao ?)

Hãy nhớ rằng
- Đừng bao giờ giả định rằng giả định của bạn là đúng
- Không bao giờ có câu trả lời chính xác cả vì vấn đề của một start-up trẻ sẽ khác so với một service có cả triệu người dùng
- Tương tác để interviewer biết mình đang nghĩ gì
