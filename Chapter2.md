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

#### Web Socket

User sẽ gửi và nhận các thông tin liên quan đến location thông qua socket, do đó ta cần những APIs sau đây:

**1. Periodic location update:**

- Request: client gửi kinh độ, vĩ độ & timestamp
- Response: không có gì

**2. Client receive location update:**

- Dữ liệu gửi đi: dữ liệu về location của bạn bè & timestamp

**3. Websocket initialization:**

- Request: Client gửi kinh độ, vĩ độ và timestamp
- Response: Client nhận về dữ liệu liên quan đến location của bạn bè

**4. Theo dõi bạn bè mới:**

- Request: Websocket server gửi friend ID
- Response: toạ độ của bạn bè & timestamp

**5. Bỏ theo dõi bạn bè:**

- Request: Websocket server gửi friend ID
- Response: không có gì

#### HTTP requests

API server xử lí các chức năng đơn giản như:

- Thêm/ xoá bạn bè
- Cập nhật user profile
- ...

#### Data model

##### Location cache

Lưu thông tin liên quan đến vị trí mới nhất của các active user có bật tính tăng "near by friend". Thông tin về location sẽ được lưu trong Redis.
Key/ value trong redis sẽ được lưu như sau:

![Screen Shot 2022-12-26 at 17 57 42](https://user-images.githubusercontent.com/15076665/209528597-143aa8f1-aedc-4037-8316-a09bc6835f42.png)

##### Lí do không sử dụng DB để lưu dữ liệu về vị trí

Thông tin về vị trí chỉ đơn thuần là "một vị trí" cho từng user nên sẽ tiện hơn nếu lưu vào Redis do thao tác "Đọc" & "Ghi" với Redis thường rất nhanh, ngoài ra Redis hỗ trợ TTL cho phép chúng ta bỏ đi dữ liệu của các user không còn active nữa.

Hơn thế nữa kể cả khi Redis bị "sập" thì ta có thể thay thế nó bằng một instance mới và "rỗng". Sau đó dữ liệu về location sẽ được "lấp đầy" bằng cách stream lại dữ liệu liên quan đến vị trí của người dùng. Việc redis bị "sập" như trên sẽ làm ảnh hưởng đến việc người dùng nhận được "location update" từ bạn bè do quá trình "cache warm", tuy nhiên điều này là chấp nhận được.

##### Location history database

Lưu thông tin lịch sử về vị trí của người dùng, schema trông sẽ như sau:

![Screen Shot 2022-12-26 at 21 42 21](https://user-images.githubusercontent.com/15076665/209550303-ecb6216a-5ce6-460f-8686-55971e9106fb.png)

Ta cần một database có thể xử lí các tác vụ ghi nặng như thế này. Casandra là một ứng cử viên nặng kí, tuy nhiên ta cũng có thể sử dụng Relational Database. Tuy nhiên historical data sẽ không thể "được đựng vừa" vào trong một database duy nhất, do đó ta cần sharding (có thể sử dụng user_id để tiến hành sharding). Việc sharding sẽ đảm bảo tải sẽ được phân bổ đều tới mọi shards.

## Bước 3: Design Deep Dive

### API server có thể scale lên như thế nào ?

Các API servers của ta ở đây đều là stateless nên việc auto-scale có thể dựa theo:

- CPU load
- CPU usage
- I/O

### Websocket server sẽ scale như thế nào ?

Websocket server là stateful server do đó ta cần cẩn thận khi tiến hành auto-scale. Giả sử với việc loại bỏ đi một websocket server hiện có. Trước hết ta phải "đánh dấu" rằng mọi server đều có thể "bị gỡ bỏ". Khi một server đi đến trạng thái "đang được gỡ bỏ" thì mọi kết nối sẽ **KHÔNG ĐƯỢC ĐIỀU PHỐI** tới nó. Khi mọi kết nối tới server đó bị ngắt thì server đó sẽ bị loại bỏ.

Việc release các Stateful server cũng đòi hỏi sự cẩn thận tương tự. Tuy nhiên các cloud load balancer đảm nhận rất tốt nhiệm vụ này.

### Client initialization

Khi mobile client khởi động, nó sẽ tạo một socket connection "vĩnh cửu" tới một websocket server instance. Các ngôn ngữ hiện đại cho phép khả năng maintain "nhiều" socket connection cùng lúc (với chi phí bộ nhớ nhỏ nhất).

Khi websocket connection được khởi tạo xong, client sẽ gửi thông tin về location tới server, sau đó websocket server handler sẽ thực hiện các tasks sau:

1. Cập nhật thông tin về vị trí của user trong cache.
2. Lưu thông tin về vị trí trong một biến bên của connection handler cho mục đích tính toán sau đó.
3. Load toàn bộ bạn bè của user từ DB.
4. Tiến hành lấy về thông tin về vị trí của bạn bè từ cache bằng một lô (batch) các fetch request. Với những bạn bè ở trạng thái "inactive" hoặc do TTL của cache nên thông tin về vị trí đã không còn trong cache thì sẽ không được trả về.
5. Sau khi lấy về thông tin liên quan đến vị trí của bạn bè, server sẽ tính toán khoảng cách từ người dùng đến vị trí đó, nếu nó nằm trong "bán kính tìm kiếm - search radius" thì thông tin về `user profile`, `update timestamp`, ... sẽ được trả về cho user thông qua websoket connection.
6. Với mỗi bạn bè, server sẽ tiến hành subscribe các Redis pub/sub channel (kể cả inactive user - do việc tạo channel không tốn quá nhiều chi phí cũng như subsribe Redis pub/sub channel của inactive user cũng không tốn quá nhiều I/O và CPU)
7. Gửi location của user cho Redis pub/sub channel của user.

### User Database

Chỉ có 2 bảng chính là:

- User profile
- Friendship

Tuy nhiên với lượng dữ liệu thì một instance duy nhất là không đủ nên ta cần sharding dựa theo userID (cho mục đích horizontal scaling).
Dữ liệu của `user database` sẽ được sử dụng bởi **Internal API**. Websocket server sẽ **THÔNG QUA** internal API để lấy dữ liệu thay vì truy vấn trực tiếp vào database. Tuy nhiên việc truy vấn trực tiếp hay gián tiếp qua API cũng không khác nhau nhiều về mặt hiệu năng.

### Location Cache

Với việc sử dụng Redis cộng với việc thiết lập TTL cho từng key, ta cần cập nhật TTL mỗi khi location được cập nhật, dẫn đến việc tiêu thụ một lượng bộ nhớ khá lớn
