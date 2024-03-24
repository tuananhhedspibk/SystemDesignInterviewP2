# Chương 8: Distributed Email Service

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Một email service hiện đại là một hệ thống phức tạp với nhiều tính năng. Nên việc thiết kế nó trong vòng 45 phút là điều không thể. Do đó việc "bó hẹp" phạm vi thiết kế là rất quan trọng.

### Functional requirements

- Số lượng người dùng: 1 tỷ
- Những tính năng quan trọng có thể kể ra như:

  - Gửi và nhận email.
  - Lấy về toàn bộ email.
  - Lọc email (đã đọc và chưa đọc).
  - Tìm email theo subject, người gửi, body.
  - Anti-spam và anti-virus.

- Một cách truyền thống, users kết nối với email servers thông qua các phương thức native của client như `SMTP`, `POP` hay `IMAP`. Tuy truyền thống nhưng chúng vẫn được sử dụng phổ biến hiện nay. Nhưng trong lần phỏng vấn này, chúng ta chỉ sử dụng HTTP.
- Email có thể đính kèm files, ảnh, ...

### Non-functional requirements

**Reliablity - Tính tin cậy**. Email không được phép bị mất.

**Availability - Tính sẵn có**. Email và user data nên được replicate trên nhiều nodes một cách tự động. Ngoài ra, hệ thống cũng cần tiếp tục hoạt động kể cả khi một phần bị lỗi.

**Scalability - Tính mở rộng**. Khi số lượng users tăng thì hệ thống nên có khả năng xử lí một số lượng users đủ lớn. Hiệu năng của hệ thống không nên đi xuống khi số lượng users và emails tăng.

**Flexibility & extensibility - tính uyển chuyển & dễ mở rộng**. Flexible/ Extensible system cho phép chúng ta có thể thêm các tính tăng mới, cải thiện hiệu năng một cách dễ dàng. Do các email protocol truyền thống như `IMAP`, `POP` đều có những tính năng rất hạn chế. Do đó chúng ta cần custom các protocols này để có thể đáp ứng tính uyển chuyển cũng như khả năng mở rộng của hệ thống.

### Back-of-the-envelope estimation

Hệ thống chúng ta có 1 tỉ users.

Giả sử số lượng email trung bình một người gửi trong 1 ngày là 10. QPS sẽ là `10^9 * 10 / 10^5 = 100,000`.

Giả sử một người sẽ nhận trung bình 40 emails trong ngày. Kích cỡ của email metadata là `50KB`. Metadata sẽ được hiểu là mọi thứ liên quan đến email (kể cả attachment files).

Giả sử metadata sẽ được lưu trong DB, việc lưu dữ liệu trong 1 năm cần: `1 tỉ user * 40 emails / ngày * 365 ngày * 50KB = 730 PB`

Giả sử 20% emails bao gồm attachment và attachment size trung bình là 500KB.

Lượng dữ liệu lưu trữ cho attachment là `1 tỉ user * 40 emails / ngày * 365 ngày * 20% * 500KB = 1460 PB`

Qua việc ước tính như trên, ta có thể thấy rằng lượng dữ liệu cần phải lưu là rất lớn, do đó chúng ta cần một giải pháp `distributed database`

## Bước 2: High-level design

### Email knowledge 101

#### Email protocols

**SMTP - Simple Mail Transfer Protocol**. Là phương thức dùng để **gửi** email từ email servers.

**POP - Post Office Protocol**. Là phương thức dùng để **nhận email** từ email servers. Sau khi email được downloaded, chúng sẽ được xoá trên server và do đó ta chỉ có thể truy cập được email từ duy nhất một thiết bị mà thôi. POP yêu cầu client download toàn bộ email do đó sẽ làm tăng thời gian download đặc biệt với các emails có attachment lớn.

**IMAP - Internet Mail Access Protocol**.
