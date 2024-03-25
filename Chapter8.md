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

**SMTP - Simple Mail Transfer Protocol**. Là giao thức dùng để **gửi** email từ email servers.

**POP - Post Office Protocol**. Là giao thức dùng để **nhận email** từ email servers. Sau khi email được downloaded, chúng sẽ được xoá trên server và do đó ta chỉ có thể truy cập được email từ duy nhất một thiết bị mà thôi. POP yêu cầu client download toàn bộ email do đó sẽ làm tăng thời gian download đặc biệt với các emails có attachment lớn.

**IMAP - Internet Mail Access Protocol**. Đây cũng là một giao thức nhận email. IMAP chỉ download email khi chúng ta click vào và nó không xoá email trên server, từ đó cho phép chúng ta có thể truy cập vào email trên nhiều thiết bị. Giao thức này cũng hoạt động tốt khi kết nối bị chậm do nó chỉ download email header cho đến khi email được mở ra.

**HTTPS**. Không phải là mail protocol nhưng cho phép người dùng kết nối đến web-based email.

#### DNS

DNS server được sử dụng để tìm `mail exchanger record - MX record`

![IMG_2765](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/de9f9a1e-dd26-428a-84e1-11976c1607bf)

Nếu gõ dns lookup cho `gmail.com` bằng command line, kết quả nhận được sẽ như hình trên. Mail server với priority thấp sẽ được ưu tiên tham chiếu hơn.

#### Attachment

Thường được mã hoá bằng Base64-encoding.

### Mail servers truyền thống

Các mail servers truyền thống thường chỉ hoạt động trên một server mà thôi, ngoài ra chúng chỉ phục vụ một số lượng user hữu hạn.

#### Kiến trúc mail server truyền thống

![Screenshot 2024-03-24 at 12 04 59](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/4cb5f821-b0ff-45e3-aefe-927046a401d0)

1. User A vào email client, gửi một email đến Outlook mail server. Giao thức tương tác giữa Outlook mail server và client sẽ là SMTP.
2. Outlook mail server sẽ queries DNS để tìm địa chỉ SMTP server của nơi nhận. Trong trường hợp này là Gmail SMTP server. Sau đó nó sẽ chuyển email sang cho Gmail mail server, giao thức tương tác giữa 2 email servers này cũng là SMTP.
3. Gmail server lưu email.
4. Gmail client sẽ lấy về các emails mới thông qua IMAP/POP server giúp user B nhìn thấy email từ user A.

### Storage

Ở các mail server truyền thống, emails được lưu trong local file directories và mỗi mail sẽ được lưu trong file với tên riêng. Maildir là cách quen thuộc để lưu mail

![Screenshot 2024-03-25 at 23 07 57](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/da6943d5-cda9-4850-b428-1b91f0bd513f)

File directories hoạt động tốt khi số lượng user nhỏ, thế nhưng nếu số lượng email lên đến cả tỉ thì đây sẽ là một giải pháp rất tồi.tồi

Do số lượng file tăng cũng như cấu trúc file trở nên phức tạp hơn nên dẫn đến tình trạng disk I/O bị bottleneck. Bản thân việc sử dụng local directories cũng không đáp ứng được yêu cầu về tính sẵn có và độ tin cậy do ổ đĩa có thể bị quá tải và dẫn đến việc server bị sập.

Tính năng email đã phát triển hơn rất nhiều, thay vì chỉ đơn thuần là text-based giờ đây ta còn có multimedia hay threading, ... Không những thế những email protocols cũ như POP, IMAP, SMTP không được thiết kế để hỗ trợ cả tỉ users.

### Distributed mail servers

Distributed mail servers được thiết kế để hỗ trợ các use-cases hiện đại và giải quyết vấn đề về scaling cũng như khả năng phục hồi của hệ thống.

#### Email APIs

Email APIs có thể rất khác với các mail clients khác nhau cũng như các vị trí khác nhau trong vòng đời của email. Ví dụ:

- SMTP/ POP/ IMAP APIs cho native mobile clients.
- Tương tác SMTP giữa sender server và receiver server.
- RESTful API trên HTTP cho việc tương tác giữa web-based email applications.

Trong phạm vi cuốn sách này, chúng ta chỉ tập trung vào HTTP protocol.

##### 1. Endpoint: POST /v1/messages

Gửi message đến người nhận trong `To`, `Cc` hoặc `Bcc` headers.

##### 2. Endpoint: GET /v1/folders

Trả về toàn bộ các folders của email account.

Response:

```json
[
  {
    id: "string"      // Unique folder identifier
    name: "string"    // Tên của folder
                      // Default folders có thể là All, Archive, Drát, Flagged, Junk, Sent, Trash
    user_id: "string" // Account owner
  }
]
```

##### 3. Endpoint: GET /v1/folers/{:folder_id}/messages

Trả về mọi messages trong foler (đây là phiên bản đơn giản nhưng trong thực tế cần có pagination).

##### 4. Endpoint: GET /v1/messages/{:message_id}

Trả về thông tin chi tiết liên quan đến message.

Messages là core building blocks của email app, nó bao gồm các thông tin về người gửi, nhận, message subject, body, ...

#### Distributed mail server architecture

Setup một email server để nó chạy được với một số lượng hữu hạn users không phải là một việc khó thế nhưng việc setup trên nhiều máy lại phức tạp hơn nhiều do chúng ta phải cân nhắc đến yếu tố đồng bộ hoá dữ liệu giữa các servers. Ngoài ra việc lọc mail spam cũng rất khó.

![Screenshot 2024-03-26 at 8 11 01](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/d8b0936b-72d2-4a69-86a6-1cacebf06d7f)

Hãy cùng xem xét chi tiết về các component trong thiết kế trên.

**Webmail**. User sử dụng browser để gửi và nhận email.

**Web servers**. Public-facing request/ response services, dùng cho các tính năng như login, signup, ... Trong thiết kế của chúng ta, mọi email API requests (gửi mail, load mail folders, load mails trong foler) đều đi qua web server.

**Real-time servers**. Có chức năng pushing email update đến cho client real-time. Real-time servers là stateful servers do chúng cần maintain các persistent connection.
