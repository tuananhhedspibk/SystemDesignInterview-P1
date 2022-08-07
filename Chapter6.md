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
