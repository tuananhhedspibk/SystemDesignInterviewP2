# Chương 2: Tìm những người bạn gần mình nhất

Bài toán trong chương này sẽ khác so với bài toán trong chương trước. Khác là ở chỗ vị trí của các businesses sẽ không thay đổi theo thời gian (real-time), còn vị trí của người dùng thì ngược lại.

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Dưới đây là những câu hỏi cần được hỏi để xác nhận với interviewer:

- Vị trí thế nào được coi là "gần" ? (5 miles đổ lại)
- Liệu rằng có thể tính khoảng cách giữa các users theo "đường chim bay" ? (Có thể)
- Bao nhiêu người sử dụng app, liệu có thể giả định rằng có 1 tỉ người dùng và chỉ có 10% trong số đó là sử dụng chức năng "tìm bạn bè gần đây" ? (Có thể cho là như vậy)
- Có cần lưu location history? (Có, nó có thể sẽ hữu ích cho các bài toán ML sau này)
- Liệu có thể giả định rằng các users inactive trong vòng khoảng 10 phút sẽ bị biến mất khỏi "nearby friend list" hoặc ta có thể hiển thị vị trí cuối cùng của user? (Có thể giả định rằng inactive user sẽ biến mất khỏi list)
- Không cần cân nhắc đến private privacy ở thời điểm hiện tại

### Yêu cầu chức năng

- User có thể nhìn thấy danh sách những bạn bè gần mình trên mobile app cũng như nhìn thấy thời gian cập nhật vị trí lần cuối cùng của họ
- Danh sách bạn bè gấn nhất nên được cập nhật sau một vài giây

### Yêu cầu khác ngoài chức năng

- Low latency
- Reliability: có thể chấp nhận một vài data point bị mất.
- Tính thống nhất: không yêu cầu tính thống nhất quá cao.

### Back-of-the-envelope estimation

Dưới đây là các giả định:

- Bạn bè được coi là "gần" nếu nằm trong phạm vi 5 mile tính từ phía user.
- Location refresh 30s một lần, tốc độ di chuyển trung bình của con người là (3 - 4 mile trên giờ), nên khoảng cách di chuyển trong vòng 30s có thể coi như không đáng kể.
- Trung bình, 100 triệu người sử dụng tính năng "tìm bạn bè gần nhất" mỗi ngày.
- Các users sử dụng cùng 1 lúc là 10% DAU = 10% * 100 triệu = 10 triệu.
- Trung bình 1 user có 400 bạn bè và họ đều sử dụng tính năng "tìm bạn bè gần nhất".
- App hiển thị 20 người bạn gần nhất trên mỗi page (con số này có thể tăng lên tuỳ yêu cầu).
**QPS = 100 triệu x 10% / 30 =~ 334,000**

## Bước 2: High-level design

### High-level design

Baì toán ở đây có thể trở thành việc user sẽ nhận các message update location từ những người bạn gần nhất. Có thể triển khai bằng peer-to-peer connection, để từ đó có thể tạo ra kết nối lâu dài và bền vững giữa các users

![Screen Shot 2022-12-05 at 7 57 41](https://user-images.githubusercontent.com/15076665/205521215-b8587822-ca21-40b2-a5a6-d92f86d9c538.png)

Cách làm này thiếu đi tính thực tế do kết nối giữa các mobile devices thường dễ bị ngắt quãng. Tuy nhiên nó cũng mở ra một hướng đi mới như sau:

![Screen Shot 2022-12-05 at 8 11 30](https://user-images.githubusercontent.com/15076665/205521357-961c239a-cfc2-4f2f-8926-594e4f774e9f.png)

Với thiết kế trên, backend (có thể gọi là shared backendd) sẽ có những vai trò sau:sau

- Nhận về toàn bộ các `location updates` từ tất cả các devices.
- Với mỗi `location update`, tìm các `active friends` nên nhận các location update trên và forward chúng đến device của friend.
- Nếu khoảng cách giữa 2 users vượt quá một ngưỡng cho trước, backend sẽ không tiến hành forward location update nữa.

Nghe thì có vẻ đơn giản nhưng nếu scale lớn thì sẽ gặp rất nhiều vấn đề. Ta lấy ví dụ khi hệ thống có 10 triệu active user, giả sử cứ 30s thì user sẽ tiến hành update location một lần, vậy là trong `1s` sẽ có `10 triệu / 30 ~ 334K` requests update location. Ta có thể giả sử rằng mỗi user sẽ có 400 friends và chỉ `10%` trong số đó là active thì ta sẽ có cả thảy `334K * 400 * 10% ~ 14 triệu` request trên `1s`. Đây **KHÔNG PHẢI LÀ MỘT CON SỐ NHỎ**.

### Proposed design

Đầu tiên ta sẽ đi vào design với scale nhỏ

![Screen Shot 2022-12-13 at 21 19 26](https://user-images.githubusercontent.com/15076665/207317017-9457ec41-b2ac-4246-94df-8e7c5c4ffa2e.png)

**RESTful API server** (stateless server) sẽ tiến hành xử lí các tasks phụ như :

- Thêm, bớt bạn bè
- Update user profile
- ...

**WebSocket server** đây là tập hợp các `stateful servers`, mỗi client sẽ duy trì một kết nối tới một stateful server này nhằm mục đích cập nhật user location một cách "gần như realtime".

**Redis location cache**, redis sẽ được sử dụng để lưu location gần nhất của các active users. Mỗi entry của cache đều có TTL riêng, khi TTL hết hạn thì user không còn active và dữ liệu liên quan đến user location sẽ bị loại bỏ khỏi cache. Cứ mỗi lần cập nhật dữ liệu trong cache thì TTL cũng sẽ được cập nhật theo.

**Redis Pub/Sub Server** sẽ đóng vai trò như các message bus, channel trong Redis pub/sub khá dễ dàng để tạo.

![Screen Shot 2022-12-13 at 21 35 50](https://user-images.githubusercontent.com/15076665/207319733-fb91f2a5-6f02-447b-9af9-0b46d09e8eae.png)

Location update được nhận thông qua WebSocket server được publish đến chính channel của user đó, friends của user (đóng vai trò làm subscriber của channel) sẽ nhận được các event update location này, lúc đó các WebSocket handler sẽ được gọi để tính toán lại khoảng cách của user với các subscriber, nếu nằm trong bán kính ngưỡng thì thông tin mới về location sẽ được gửi qua kết nối WebSocket tới các friends của user.

### Cập nhật location định kì

![Screen Shot 2022-12-13 at 21 52 54](https://user-images.githubusercontent.com/15076665/207323275-c37b7e00-ff38-4957-ab3c-c487ffa7c008.png)

Trên đây là flow cập nhật user location.

1. User gửi thông tin về location thông qua WS đến LB
2. LB forward đến cho WS server
3. Thông tin về user location được cập nhật trong DB
4. Thông tin về user location cũng được cập nhật trong cache (khi cập nhật cache thì TTL cũng được cập nhật theo)
5. WS server sẽ publish message cập nhật location của user đến **chính channel của user** trong `Redis pub/sub` (các thao tác 3 -> 5 có thể được thực hiện song song nhau).
6. Khi Redis pub/sub nhận được message cập nhật location nó sẽ broadcast đến mọi subsribers (WebSocket connection handlers)
7. Khi nhận được message cập nhật location, WebSocket server sẽ tính toán lại khoảng cách giữa user (phía publish message) và các subscriber
8. Nếu khoảng cách nằm trong ngưỡng cho phép thì WbeSocket sẽ gửi thông tin về location mới đến cho các users tương ứng.

Cụ thể hơn về flow update location sẽ như sau:

![Screen Shot 2022-12-13 at 22 13 19](https://user-images.githubusercontent.com/15076665/207335164-c376cd23-9b2c-435e-a355-a098a6d19082.png)

Ở đây ta giả sử:

- User 1 là bạn với User 2,3,4
- User 5 là bạn với User 4,6

1. Khi location của User 1 thay đổi, thông tin về location mới sẽ được gửi qua WS connection tới WS server tương ứng
2. WS server sẽ publish message này đến User 1 channel trong Redis pub/sub
3. Channel sẽ broadcast message đến các subsribers (WS connection handler tương ứng với bạn bè của User 1)
4. Nếu khoảng cách từ User thay đổi location tới subscriber không vượt quá ngưỡng cho trước thì thông tin về location mới của User 1 sẽ được gửi tới cho subscriber.

Flow trên sẽ được lặp đi lặp lại cho mọi subcribers của channel, nên ở đây ta giả sử 1 user có 400 friends, trung bình có 10% là online và gần với user nên với 1 update location message thì sẽ có `400 * 10% = 40` updates tương ứng.

### API design

