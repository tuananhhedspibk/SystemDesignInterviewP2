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

[Screen Shot 2022-12-05 at 7 57 41](https://user-images.githubusercontent.com/15076665/205521215-b8587822-ca21-40b2-a5a6-d92f86d9c538.png)

Các làm này thiếu đi tính thực tế do kết nối giữa các mobile devices thường dễ bị ngắt quãng. Tuy nhiên nó cũng mở ra một hướng đi mới như sau:

[Screen Shot 2022-12-05 at 8 11 30](https://user-images.githubusercontent.com/15076665/205521357-961c239a-cfc2-4f2f-8926-594e4f774e9f.png)
