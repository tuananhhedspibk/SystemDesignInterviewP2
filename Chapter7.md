# Hotel Reservation System

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Hệ thống đặt lịch phòng khách sạn thường khá phức tạp và đa dạng tuỳ theo business model. Do đó trước khi thiết kế, ta cần làm rõ yêu cầu với interviewer.

- Hệ thống phải phục vụ cho một chuỗi khách sạn với 5000 khách sạn và tổng số là 1 triệu phòng.
- Người dùng sẽ phải trả toàn bộ chi phí khi tiến hành đặt phòng.
- Người dùng có thể đặt phòng qua web hoặc app.
- Người dùng có thể cancel lịch đặt phòng của họ.
- Hệ thống sẽ đáp ứng 10% overbooking (tức là khách sạn sẽ cho phép thuê "nhiều phòng" hơn số phòng họ hiện có, mục đích ở đây là xử lí những case khách hàng cancel lịch hẹn).
- Do thời gian thiết kế có hạn nên sẽ chỉ tập trung vào những tính năng như phía dưới đây:

1. Hotel-related page.
2. Hotel room-related detail page.
3. Đặt phòng.
4. Admin panel để thêm/ sửa/ xoá thông tin phòng.
5. Hỗ trợ overbooking.

- Giá thuê phòng sẽ thay đổi tuỳ theo ngày.

### Non-functional requirements

- Hỗ trợ high-concurrency. Vào các mùa "cao điểm" du lịch hoặc sự kiện lớn, các khách sạn nổi tiếng sẽ có nhiều khách hàng cùng book phòng.
- Độ trễ vừa phải. Sẽ là lí tưởng nếu hệ thống có thể hồi đáp lại phía user nhanh nhất có thể thế nhưng độ trễ vài giây cũng có thể được coi là chấp nhận được.

### Back-of-the-envelope estimation

- 5000 khách sạn và tổng số 1 triệu phòng.
- Giả sử 70% phòng đã được đặt và thời gian sử dụng trung bình là 3 ngày.
- Lượng đặt phòng theo ngày = (1 triệu \* 0.7 / 3) = 233,333 ~ 240,000
- Lượng đặt phòng theo giây: 240,000 / 10^5 = 3 (TPS không quá cao).

Giờ chúng ta sẽ tính toán QPS cho mọi pages. Có 3 bước trong follow điển hình của người dùng như sau:

1. Xem thông tin khách sạn/ phòng (query).
2. Xem trang đặt phòng. Người dùng có thể xác nhận các thông tin chi tiết về việc đặt phòng như (ngày, số lượng khách, payment infor) (query).
3. Đặt phòng, user click vào nút đặt phòng (transaction).

Ta giả sử khoảng 10% users sẽ đi đến bước tiếp theo và 90% users sẽ bỏ giữa chừng flow nói trên.

![Screenshot 2024-03-09 at 13 38 52](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/661f9167-0a28-4244-a10e-ea91e31fa2b5)

Hình trên đây sẽ cho thấy bản estimate "thô" về QPS, ở đây ta lần ngược từ giá trị **Lượng đặt phòng theo giây** ở trên.

## Bước 2: High-level design

### API design

Với hệ thống đặt phòng này, ta sẽ thiết kế theo hướng RESTful API.

![Screenshot 2024-03-09 at 16 28 13](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/1e53d81b-8114-4f23-9f6e-34fe70ba1c55)

![Screenshot 2024-03-09 at 16 29 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/4fdedac9-0acf-4ce8-aa14-d7f6a38f74b9)

![Screenshot 2024-03-09 at 16 29 42](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/7900d4a0-886c-42ed-b20d-87a4a020a80e)

Chú ý rằng `reservation ID` sẽ được sử dụng như idempotency key để tránh double book (ở đây mang ý nghĩa là nhiều lần đặt cùng 1 phòng tại cùng 1 thời điểm).

### Data model

Với hệ thống đặt phòng khách sạn, chúng ta cần support những câu queries sau:

- Query 1: Xem thông tin chi tiết về khách sạn.
- Query 2: Tìm các loại phòng hiện có trong một khoảng thời gian nhất định.
- Query 3: Đặt phòng.
- Query 4: Tìm lịch sử đặt phòng.

Theo như yêu cầu ở phần **back-of-envelope estimate**, ta thấy rằng quy mô hệ thống không quá lớn, chỉ cần đáp ứng được những đợt cao điểm du lịch là đủ, do đó ta sẽ sử dụng RDB ở đây với những lí do như sau:

- **RDB hoạt động tốt với read-heavy và write less** do trong thực tế số lượng những người thực sự đặt phòng ít hơn rất nhiều so với những người xem thông tin về khách sạn và phòng (NoSQL hoạt động tốt hơn với các ứng dụng ghi nhiều hơn đọc).
- RDB cung cấp ACID (atomicity, consitency, isolation, durability), ACID sẽ phát huy sức mạnh với các hệ thống reservation kiểu này do nó xử lí tốt các vấn đề như **negative balance**, **double charge**, **double reservations**, ... ACID sẽ giúp cho application code đơn giản hơn rất nhiều.
- RDB cũng dễ dàng giúp mô hình hoá dữ liệu. Do cấu trúc của business data đã rất rõ ràng và mối quan hệ giữa các entities (hotel, room, room_type, ...) cũng ổn định,
