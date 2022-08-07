# Scale From Zero To Millions Of Users

Chúng ta nên bắt đầu từ những thứ nhỏ nhất. Khi bắt đầu triển khai một hệ thống, hãy bắt đầu chỉ với 1 server duy nhất dùng chung cho cả API + DB.

Khi số lượng người dùng tăng dần lên, hãy nghĩ tới chuyện tách riêng chúng thành 2 servers khác nhau.

Kiến trúc ban đầu:

![img](https://user-images.githubusercontent.com/15076665/170048273-5c320fa4-f891-411b-b165-e0da3228c22e.png)

Kiến trúc sau khi đã phân tách server:

![img](https://user-images.githubusercontent.com/15076665/170048649-a568908d-662e-43aa-9cdb-93e3090a7ec1.png)

**Chúng ta sẽ chọn database nào ?**

Về cơ bản khi số lượng truy cập vào DB tăng dần, ta sẽ chia thành 2 cluster DB khác nhau:

- Master: chuyên ghi (UPDATE, INSERT, DELETE)
- Slave: chuyên đọc (READ ONLY)

![img](https://user-images.githubusercontent.com/15076665/170053779-3f4cf77b-adb2-47ab-a08d-46b05a3c490a.png)

Mô hình `Master/ Slave` này sẽ giúp cải thiện hiệu năng do:

- Tăng tính tin cậy: khi 1 DB down thì vẫn sẽ có các DB khác hoạt động
- Tính sẵn sàng cao: do dữ liệu luôn được sao lưu ở nhiều DB nên khi 1 trong số đó go offline thì ứng dụng vẫn luôn có thể phục vụ người dùng với dữ liệu mới nhất.

Nếu trong TH Slave có vấn đề, sẽ có các slaves khác thay thế. Còn nếu master có vấn đề thì một trong số những slave hiện tại sẽ trở thành master mới.

![img](https://user-images.githubusercontent.com/15076665/170054601-f6dff79f-ef4e-45f1-a545-6859cddcac5d.png)

Về loại DB ta có thể chia thành 2 nhóm chính:

- Relational Database: RDB
- Non-relational Database: NoSQL

**Non-relational Database** sẽ là sự lựa chọn hoàn hảo nếu:

- Ứng dụng yêu cầu độ trễ cực thấp
- Dữ liệu không có cấu trúc hoặc quan hệ
- Chỉ cần serialize hoặc de-serialize data (JSON, XML, YAML)
- Cần lưu một số lượng lớn dữ liệu

**Vertical Scaling** vs **Horizontal Scaling**

1. Vertical Scaling
Hay còn gọi là "scale up" - tăng thêm RAM, CPU cho server
2. Horizontal Scaling
Tăng số lượng các servers lên

> Vertical Scaling sẽ bị hạn chế do ta không thể tăng một cách không có giới hạn tài nguyên của server như CPU hay RAM.

Hơn nữa vertical scaling không có **tính chịu lỗi** hoặc **tính dự phòng** do khi 1 server down thì hệ thống của bạn cũng sẽ sập theo.

Tuy nhiên nếu trong trường hợp server bị offline thì hệ thống của bạn cũng sẽ bị "treo" theo, ngoài ra khi số lượng access của user quá nhiều, sẽ dẫn đến tình trạng người dùng sẽ phải trải nghiệm kết nối chậm tới server của bạn. Từ đó dẫn đến nhu cầu cần phải có **load balancer**

> Load balancer sẽ đứng giữa user và server

Như hình dưới đây:

![img](https://user-images.githubusercontent.com/15076665/170052060-018ffa02-3dac-4e8d-90ac-dd50e98e5a9e.png)

Lúc này user sẽ tương tác trực tiếp với load balancer, load balancer sẽ tương tác với các server thông qua private IP của sever

Ngoài việc tránh cho user tương tác trực tiếp với server, load balancer còn có chức năng phân tải sao cho số lượng request đến các server được cân bằng.

Sau khi đã có cái nhìn về những phần core của hệ thống, chúng ta sẽ tiến hành cải thiện tốc độ xử lí và phản hồi của hệ thống với `Cache layer` và `CDN`

**Cache**
Là một dạng bộ nhớ temporary. Thường sẽ lưu các kết quả hay được truy vấn hoặc các kết quả của các truy vấn nặng.

**Cache tier**
Là một tầng lưu bộ nhớ tạm thời. Mô hình thực tế sẽ như sau:

![img](https://user-images.githubusercontent.com/15076665/170057108-afa30450-9731-44dc-ba57-ed87bf68768c.png)

Tương tác với các cache servers thường khá đơn giản vì chúng cũng đã cung cấp API cho các ngôn ngữ lập trình khác nhau.

![img](https://user-images.githubusercontent.com/15076665/170058277-99f09241-6476-4b6e-8fda-4e2496489c56.png)

Hãy cân nhắc sử dụng cache trong trường hợp sau:

- Dữ liệu được đọc nhiều hơn là được ghi

Dữ liệu cache được lưu trong một bộ nhớ tạm nên cache server không phải là một công cụ lí tưởng để lưu trữ data lâu dài. Giả sử trong trường hợp server bị restart, mọi dữ liệu trong bộ nhớ sẽ bị mất. Do đó các dữ liệu quan trọng nên được lưu trong bộ nhớ lâu dài (persistent data store).

Ngoài ra cũng nên cân nhắc đến `Expired Policy`. Dữ liệu lưu trong cache nên bị xoá đi sau một khoảng thời gian `expired` nhất định, nếu không xoá đi thì cache server sẽ bị phình to. Tuy nhiên không nên đặt expire time quá ngắn vì khi đó server sẽ phải truy vấn dữ liệu thường xuyên hơn.

Ngược lại cũng không nên thiết lập `expired time` quá lâu vì sẽ khiến cho dữ liệu cache bị cũ.

Một vấn đề khác cần cân nhắc đó là tính "nhất quán" về giữ liệu của cache và DB. Hơn nữa khi triển khai ở nhiều regions khác nhau thì việc đồng bộ hoá dữ liệu là một vấn đề khá đau đầu.

Giảm thiểu lỗi cũng là một điều cần cân nhắc: một cache server duy nhất sẽ khiến cho SPOF (Single Point Of Failure) trở nên tiềm tàng hơn, vậy nên multiple cache servers với các data centers khác nhau sẽ giúp ngăn ngừa SPOF

**Eviction Policy**: Khi cache đầy, để thêm dữ liệu mới vào cache ta cần phải bỏ đi những giữ liệu đã cũ hoặc ít được sử dụng (Least-recently-used : LRU). Ngoài cách làm này ta cũng có thể cân nhắc tới chính sách FIFO - First In First Out.

CDN: dùng để truyền tải các static content đến người dùng gần nhất về mặt địa lý (CSS, images, JS).

Cơ chế làm việc của CDN:

- Khi user req đến website, CDN server gần người dùng nhất sẽ truyền tải nội dung đến cho người dùng, vậy nên CDN càng xa thì người dùng load web sẽ càng chậm.

Hình dưới đây sẽ cho thấy hiệu quả rõ rệt của việc sử dụng CDN.

![img](https://user-images.githubusercontent.com/15076665/170821877-c0e5d0af-fcd5-4b9d-8cf9-5fc848553498.png)

Nếu trường hợp dữ liệu mà user req không có trong CDN, CDN sẽ truy xuất dữ liệu từ origin, lưu lại tại CDN rồi trả về phía user.

> Khi origin trả dữ liệu về cho CDN, nó sẽ bao hàm một optional HTTP header: Time-to-live (TTL)

## Cân nhắc về việc sử dụng CDN

- Giá thành: CDN thường được vận hành bởi 1 bên thứ 3, do đó nếu dữ liệu cache không được sử dụng thường xuyên, bạn nên cân nhắc việc từ bỏ CDN để tiết kiệm chi phí.
- Thiết lập thời gian hết hạn cho cache với những content liên quan trực tiếp tới thời gian.
- CDN fallback: Bạn nên tính đến trường hợp CDN bị sập, khi đó client nên có sự nhận biết để chuyển hướng req đến origin.
- Invalidating files: bạn có thể cân nhắc việc loại bỏ các file trong CDN trước khi chúng hết hạn bằng những cách sau:

1. Sử dụng API được phát triển bởi CDN provider.
2. Versioning object bằng cách thêm `version param`  vào URL: <https://source/image.png?version=2>

Hình dưới đây là hệ thống sau khi có thêm CDN và cache.

![img](https://user-images.githubusercontent.com/15076665/170822281-d2c609db-0b10-44c8-aa02-11e9cb05e17a.png)

1. Static assets (JS, CSS, images) được cung cấp bởi CDN thay vì server
2. Database loading được cải thiện nhờ caching data.

**Stateless web tier**: server sẽ không lưu trữ `user session` mà các thông tin này sẽ được lưu trữ trong DB. Từ đó dẫn đến các servers trong một cluster có thể truy xuất `user session data` thông qua DB.

![img](https://user-images.githubusercontent.com/15076665/170822575-6c7bc95a-af46-40c5-8df0-0d3d3daca6d9.png)

Với kiến trúc stateless như trên, các req từ phía user có thể được điều hướng đến bất kì server nào do session data sẽ được lưu trong một shared DB.

Session data nên được lưu vào NoSQL database do nó có thể dễ dàng được scale.

**Stateful architecture**: server sẽ ghi nhớ thông tin về state của user như hình dưới đây.

![img](https://user-images.githubusercontent.com/15076665/170822459-06dd8202-2484-402c-9240-52900db91636.png)

Khi đó mọi req từ user A sẽ phải đến server 1 vì nếu đến server 2 thì authenticate sẽ fail, từ đó dẫn đến một vấn đề đó là mỗi req từ phía user sẽ phải được điều hướng đến một server duy nhất. Điều này có thể giải quyết nhờ load balancer, tuy nhiên việc tăng / giảm server thường xuyên cũng như quản lí server failing cũng là một vấn đề khá đau đầu.

Ngoài vấn đề về state, với những trang web hoạt động toàn cầu thì vấn đề đảm bảo UX với người dùng dựa theo vị trí địa lí của họ cũng là một điều vô cùng quan trọng, từ đó dẫn tới việc cần thiết phải cung cấp các data centers khác nhau để phục vụ người dùng.

![img](https://user-images.githubusercontent.com/15076665/170826669-79a39676-a3df-47ad-997f-e53dfcb40163.png)

Hình phía trên cho ta một cái nhìn trực quan về multiple data centers. Trong trường hợp thông thường, user sẽ được geoDNS-routed (còn gọi là geo-routed) đến data center gần nhất.

geoDNS là dịch vụ DNS map domain name với IP address dựa trên vị trí địa lí của người dùng.

Khi có một trong 2 data centers gặp sự cố, mọi req sẽ được hướng tới healthy data center như hình bên dưới.

![img](https://user-images.githubusercontent.com/15076665/170826829-547b3402-decc-4f55-8ce0-e5e43279af33.png)

Sẽ có những vấn đề sau khi ta tiến hành setup multiple data center:

- **Traffic Redirection**: cần phải điều hướng người dùng đến data center cần người dùng nhất. GeoDNS có thể được sử dụng để giải quyết vấn đề này
- **Data Synchronization**: những người dùng ở các locations khác nhau có thể sẽ được truy cập vào các DB với dữ liệu khác nhau đôi chút. Nhưng trong trường hợp một data center gặp sự cố, khi đó người dùng sẽ bị re-direct đến data center không có dữ liệu mà người dùng cần. Để giải quyết vấn đề này, ta cần đồng bộ hoá dữ liệu giữa các data centers khác nhau.
- Test and deployment: với multi data center thì việc test cũng rất quan trọng khi bạn cần test ở nhiều locations khác nhau. Automated deploy là một công cụ giúp bạn giữ được sự thống nhất giữa các data centers.

**Message queue**: là công cụ giúp ta có thể deploy các components trong hệ thống một cách độc lập. Được lưu trong bộ nhớ, hỗ trợ asynchronous communication. Nó hoạt động như một buffer và phân phối các asynchronous requests.

Kiến trúc cơ bản của một message queue rất đơn giản. Input services hay còn gọi là producer/ publisher sẽ tạo và đưa message vào queue, các services khác hay còn gọi là consumer/ subscriber sẽ lấy message từ queue ra và thực thi action được định nghĩa bởi message.

![img](https://user-images.githubusercontent.com/15076665/170827627-e89f4260-b3ea-4bb2-8090-d175a66945ba.png)

Với message queue thì `producer` vẫn có thể publish message kể cả khi `consumer` không hoạt động và ngược lại `consumer` vẫn có thể lấy message và thực thi action kể cả khi `producer` không hoạt động.

Lấy ví dụ với các tasks xử lí ảnh (zoom, crop, ...) - đây là các tasks tốn nhiều thời gian để thực hiện. Consumer có thể lấy message yêu cầu xử lí ảnh từ phía producer và thực thi task xử lí ảnh này asynchronous.

Khi message queue có hiện tượng bị đầy, các workers khác có thể được thêm vào để giảm đi độ trễ trong xử lí task. Khi queue rỗng thì số lượng các workers sẽ được giảm đi để tránh lãng phí tài nguyên.

### Logging, metrics, automation

Với các hệ thống nhỏ thì việc thiết lập một hệ thống log là một good practice nhưng không thực sự cần thiết. Thế nhưng khi service của bạn phình to và phục vụ một business lớn thì đây là điều vô cùng quan trọng trong quá trình vận hành hệ thống.

**Logging**: là việc theo dõi log của server, điều này vô cùng quan trọng để bạn có thể định danh được bug và từ đó đưa ra phương án xử lí nhanh nhất.

**Metrics**: biết được tình trạng của hệ thống, cũng như biết được xu hướng hiện tại của business. Một vài metrics sau đây là vô cùng cần thiết:

- Host level metrics: CPU, Memory, disk I/O
- Aggregate level metrics: performance của DB tier
- Key business metrics: Daily active user (DAU), retention, doanh thu.

Hình bên dưới là thiết kế cho hệ thống đã bao gồm logging và metric.

![img](https://user-images.githubusercontent.com/15076665/170850450-123ffde5-f69f-4abf-ad2b-1dd6acd3c57f.png)

Ki hệ thống trở nên lớn hơn, việc tích hợp tự động (Auto integration) là việc cần thiết để có thể:

- Cải thiện được năng suất của dev
- Giúp team phát hiện được lỗi sớm hơn

### Database Scaling

Có 2 hướng tiếp cận cho vấn đề này: **Vertical Scaling** và **Horizontal Scaling**

**Vertical Scaling** hay còn gọi là scale up, ta tiến hành bằng cách tăng tài nguyên cho DB server (CPU, RAM, DISK, ...). Có một vài công cụ RDB khá mạnh như Amazon RDB cho phép RAM tối đa lên tới 24TB. Tuy nhiên cách làm này có những hạn chế như sau:

- Tăng tài nguyên cho server khá tốn kém về chi phí
- Tăng rủi ro SPF (single point of failure)
- Việc tăng tài nguyên cho server là giới hạn

**Horizontal Scaling** còn gọi là **sharding** - là cách mà ta sẽ thêm thêm các DB servers khác.

![img](https://user-images.githubusercontent.com/15076665/170850593-f646520d-30f3-439b-9d6a-5a36f415af9e.png)

Việc chia nhỏ 1 DB to thành các shards con giúp việc quản lí dễ dàng hơn. Các shards sẽ chia sẻ cùng 1 schema nhưng data trên mỗi shard là khác nhau.

![img](https://user-images.githubusercontent.com/15076665/170850796-65c88813-9178-4ba7-8e43-9fe1e14b4984.png)

Hình bên trên mô tả một trường hợp về việc chia dữ liệu đến các shards dựa theo `user_id`, ta sẽ dựa theo `user_id % 4`. Nếu `user_id % 4 === 0`, data sẽ đi vào shard 0, tương tự với các trường hợp còn lại.

Dưới đây sẽ là kết quả:

![img](https://user-images.githubusercontent.com/15076665/170850845-8a840f1a-c930-4c32-86b0-c792cc29da5e.png)

Một trong những yếu tố cần cân nhắc khi sử dụng sharding DB đó là việc lựa chọn **Sharding Key** hay còn gọi là **Partition Key**. Sharding key sẽ quyết định một hoặc những cột nào sẽ quyết định về việc phân bổ dữ liệu vào các shard.

Sharding key cho phép các query truy xuất hoặc modify data sẽ đến đúng DB.

> Một tip khi chọn sharding key đó là chọn key có thể phân bố dữ liệu đồng đều nhất

Sharding là một lựa chọn tốt khi tiến hành scale DB nhưng nó không phải là sự lựa chọn tốt nhất, do việc sư dụng sharding có thể sẽ dẫn đến các vấn đề sau đây:

- **Resharding data**:  cần thiết với những tình huống sau

1. Một shard không thể chứa thêm dữ liệu do lượng dữ liệu đến nó tăng đột biến
2. Shard exhaustion do việc phân bổ dữ liệu không đồng đều giữa các shards. Lúc này cần cập nhật lại hàm phân bổ dữ liệu và tiến hành di chú dữ liệu giữa các shards để đạt tới trạng thái đồng đều.

- **Celebrity problem**: còn gọi là **hotspot key problem**.  Đây là tình trạng một shard bị quá tải do read operation quá nhiều (VD: với các chủ đề hot hoặc các nhân vật hot)
- **Join and de-normalization**: khi tiến hành phân bổ data ra các shards thì việc thực hiện câu lệnh `JOIN` giữa các bảng là hoàn toàn không thể. Do đó trong các trường hợp này ta không nên "chuẩn hoá DB" để câu queries có thể được thực hiện trên một bảng đơn.

Scaling system là một công việc lặp đi lặp lại vô tận khi số người dùng không ngừng tăng lên.
VD:

- Tối ưu hoá hệ thống (logic xử lí với 1,000 users sẽ khác với 1,000,000 users)
- Chia nhỏ những modules sẵn có trong hệ thống thành các modules nhỏ hơn.

Dưới đây sẽ là tổng kết cách scale out một hệ thống để phục vụ cả triệu users:

- Giữ cho web tier stateless
- Xây dựng một cơ chế chịu lỗi ở mọi tier
- Cache data nhiều nhất có thể
- Sử dụng multiple data centers
- Host static assets bằng CDN
- Scale data tier bằng sharding
- Chia nhỏ các tier
- Monitor system, sử dụng các automation tool (AUTO integration, deploy)
