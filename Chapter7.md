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
