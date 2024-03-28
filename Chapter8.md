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

**Real-time servers**. Có chức năng pushing email update đến cho client real-time. Real-time servers là stateful servers do chúng cần maintain các persistent connection. Để hỗ trợ real-time communication chúng ta có một vài options như long polling hoặc Websocket. Websocket là một giải pháp kinh điển nhưng nó có một nhược điểm đó là "xung đột" với browser. Một giải pháp khả quan nhất đó là thiết lập Websocket connection khi có thể và sử dụng long-polling như một giải pháp dự phòng.

**Metadata DB**. Đây là DB lưu mail meta-data bao gồm mail subjet, body từ người gửi cũng như đến người nhận.

**Attachment store**. Chúng ta lựa chọn S3 do S3 là một storage infrastructure có khả năng scale tốt cũng như phù hợp để lưu các files có kích cỡ lớn như images, videos, ... Attachment có thể có dung lượng lên đến 25MB. Một lựa chọn khác với NoSQL như Cassandra sẽ không phù hợp vì 2 lí do sau:

- Ngay cả khi Cassandra hỗ trợ blob type (về mặt lí thuyết dung lượng lên đến 2GB) nhưng trong thực tế thì giới hạn là < 1MB.
- Một vấn đề khác với Cassandra đo là khi đưa attachments vào nó, ta không thể sử dụng row cache như attachments chiếm một lượng lớn không gian nhớ.

**Distributed cache**. Do các emails gần đây thường được load đi load lại bởi client, việc caching các emails gần đây sẽ là một giải pháp giúp cải thiện load time. Chúng ta có thể sử dụng Redis ở đây vì nó cung cấp nhiều tính năng như lists và khả năng scale dễ dàng.

**Search store**. Là một distributed document store. Nó sử dụng cấu trúc dữ liệu `inverted index` - hỗ trợ full-text search tốc độ cao.

Về cơ bản sau khi lắp ghép các components phía trên lại, chúng ta có thể xây dựng được 2 workflows chính như sau:

- Flow gửi email.
- Flow nhận email.

#### Flow gửi email

![Screenshot 2024-03-26 at 22 57 29](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/696239f5-a548-4590-9a8b-74d0b48f288e)

1. User gửi email, request sẽ đi đến Load Balancer.
2. Load balancer đảm bảo việc số lượng request không vượt quá rate limit và điều hướng traffic đến web server.server
3. Web server sẽ đảm nhiệm:

- Email validation. Mỗi một email đến sẽ được kiểm tra dựa trên các `pre-defined rules` như email size limit.
- Kiểm tra xem domain của địa chỉ người nhận có giống người gửi hay không. Nếu giống thì web server chắc chắn email không có virus và cũng không phải là spam email. Nếu vậy, email sẽ được thêm vào "Sent Folder" của người gửi và "Inbox Folder" của người nhận. Người nhận sau đó có thể lấy về qua RESTful API và không cần đi đến bước 4.

4. Message queues

   4.1. Nếu email validate pass, email data sẽ được đưa đến `outgoing queue`, nếu attachment đi kèm với email quá lớn thì nó attachment sẽ được lưu trong `Object storage` và tham chiếu của object attachtment sẽ được đưa vào message queue trên.
   4.2. Nếu email validate failed, email sẽ được đưa vào `error queue`.

5. `SMTP outgoing worker` sẽ kéo messages từ `outgoing queue` và đảm bảo việc chúng là spam, virus free.
6. Outgoing email được lưu trong "Sent Folder" nằm trong storage layer.
7. `SMTP outgoing worker` gửi mail đến cho mail server của người nhận.

Các messages trong outgoing queue bao gồm mọi meta-data cần thiết cho việc tạo email. Distributed message queue là thành phần quan trọng cho phép xử lí email một cách bất đồng bộ. Qua đó ta có thể decoupling `SMTP outgoing worker` và `web servers`. Từ đó chúng ta có thể scale SMTP outgoing workers một các độc lập.

Chúng ta luôn xem xét size của outgoing queue rất kĩ, nếu có bất kì emails nào stuck trong đó, chúng ta cần tìm ra nguyên nhân. Dưới đây là một vài khả năng có thể xảy ra:

- Mail server của phía người nhận không hoạt động. Trong tình huống này, cách giải quyết duy nhất đó là retry. Retry với tần số theo số mũ (exponential backoff) là một sự lựa chọn phổ biến.
- Không đủ consumer để gửi mail. Trong tình huống này, ta cần tăng số lượng consumer để giảm thời gian xử lí.

#### Flow nhận email

Hình dưới đây mô tả flow nhận email.

![Screenshot 2024-03-27 at 22 03 17](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/cf3b6f4f-1261-4b91-9b21-5d11497eac12)

1. Email đến sẽ được gửi đến SMTP load balancer.
2. Load balancer sẽ phân bổ traffic đến các SMTP servers. Email acceptance policy được thiết lập và áp dụng ngay tại SMTP-connection level. Ví dụ như việc ta có thể lọc các invalid email để không phải tốn thời gian xử lí.
3. Nếu attachment quá lớn để đưa vào queue, ta có thể đưa nó vào attachment store (S3).
4. Email được đưa vào queue, việc làm này giúp decoupling queue worker và SMTP server, để từ đó ta có thể scale chúng một cách độc lập. Hơn nữa queue sẽ hoạt động giống như một buffer trong trường hợp số lượng email "bùng nổ".
5. Mail processing servers đảm nhận khá nhiều tác vụ như: lọc spam emails, ngăn chặn virus, ...
6. Mail data được lưu trong Mail storage, cache, object data store.
7. Nếu người nhận đang online, mail sẽ được push thẳng lên real-time servers.
8. Real-time servers là các WebSocket servers cho phép client nhận email mới một cách real-time.
9. Với offline user, email sẽ được lưu trong storage layer, sau đó web-client sẽ lấy về các emails mới nhất thông qua RESTful API.
10. Web server pull các emails mới nhất và trả về cho phía client.

## Bước 3: Deep-dive design

Trong phần này chúng ta sẽ đi sâu vào một vài components như sau:

- Metadata DB.
- Search
- Deliveribility
- Khả năng mở rộng

### Metadata DB

#### Đặc chưng của email metadata

- Email headers thường có kích cỡ nhỏ và hay được truy cập.
- Email body có thể nhỏ hoặc lớn nhưng chỉ được đọc một lần duy nhất.
- Các thao tác với email như đánh dấu đọc, fetching chỉ được thực hiện bởi user sở hữu email mà thôi.
- User thường chỉ đọc các dữ liệu gần đây, trên thực tế user chỉ đọc các email trong vòng 16 ngày gần nhất.
- Dữ liệu cần có độ tin cậy cao và KHÔNG ĐƯỢC PHÉP MẤT.

#### Chọn đúng DB

Với quy mô cỡ Gmail hoặc Outlook, DB system thường được custom-made để giảm đi input output per second (IOPS), đây có thể trở thành một trong những điều kiện quan trọng của hệ thống. Việc chọn DB phù hợp không phải đơn giản, đôi khi chúng ta nên xem xét các điều kiện trên các bảng trước khi đưa ra sự lựa chọn DB phù hợp nhất.

- `Relational DB`. Với DB kiểu này ta có thể đánh indexes trên mail header và body để tăng tốc độ truy vấn. RDB được tối ưu hoá cho data chunks cỡ nhỏ chứ không phải cỡ lớn. Một email điển hình thường có kích cỡ vài KB và có thể vượt quá cả trăm KB nếu có HTML bên trong, với data cỡ lớn, ta có thể sử dụng BLOB type nhưng search queries trên BLOB data type lại không phát huy tác dụng. Nên do đó các sự lựa chọn như MySQL hay PostgreSQL không phải là những sự lựa chọn tốt.
- `Distributed Object Storage`. Một giải pháp khác đó là lưu trong cloud storage như S3, đây sẽ là một giải pháp backup tốt nhưng lại không đáp ứng được các tính năng như search emails by keywords, ...
- `No SQL` là một lựa chọn tốt, bản thân Google sử dụng BigTable nhưng bản thân công nghệ này chưa được public nên cách thức email searching được triển khai vẫn là một ẩn số. Cassandra có thể được sử dụng ở đây nhưng độ phổ biến không cao.

Với các mail service có quy mô lớn, chúng ta thường sẽ phải customized database, nhưng trong phạm vi của một buổi phỏng vấn, việc thiết kế một distributed database là điều không thể, do đó ta chỉ cần liệt kê ra những đặc điểm cần có sau của DB là được:

- Single column có thể là single-digit của MB.
- Tính thống nhất về dữ liệu cao.
- Được thiết kế để giảm disk I/O.
- Tính sẵn có cao và fault-tolerant.
- Dễ dàng tạo backup.

#### Data model

Một cách để lưu dữ liệu đó là sử dụng `user_id` như partition key, nên dữ liệu của một user sẽ được lưu trong một shard.

Một nhược điểm của cách làm này đó là messages không được shared giữa các users khác nhau.

Trong lần này, primary key sẽ là tổ hợp của `partition_key` & `clustering_key`.

- `partition_key`　 có nhiệm vụ chính là phân bổ dữ liệu vào các nodes, điều chúng ta muốn ở đây đó là việc dữ liệu được phân bổ một cách đồng đều.
- `clustering_key` có nhiệm vụ sắp xếp dữ liệu trong cùng một partition.

Email service cần hỗ trợ những queries như sau ở data layer.

- Query 1: Lấy về tất cả các folders thuộc về cùng một user.
- Query 2: Hiển thị toàn bộ email trong một folder.
- Query 3: Create/ Delete/ Get một email cụ thể.
- Query 4: Lấy toàn bộ các read hoặc unread email.
- Bonus point: lấy về conversation threads.

##### Query 1: Lấy về tất cả các folders thuộc về cùng một user

`user_id` được sử dụng làm partition key, do đó mọi folders thuộc về cùng một user sẽ nằm trong cùng một partition.

![Screenshot 2024-03-27 at 22 07 31](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/6ff59c8d-7002-4ae6-bc68-ce6f836c3bdd)

##### Query 2: Hiển thị toàn bộ email trong một folder

Khi user load emails trong inbox, các emails mới nhất sẽ được hiển thị lên trước. Để lưu các emails của một user vào cùng một folder, ta cần đến composite key `<user_id, folder_id>`.

Một cột khác cũng được sử dụng ở đây đó là `email_id` (data type là `TimeUUID` sẽ được sử dụng như `clustering_key`　 cho mục đích sắp xếp).

![Screenshot 2024-03-27 at 22 09 07](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/bd463a09-5483-423b-96ee-1dd73ad34d63)

##### Query 3: Create/ Delete/ Get một email cụ thể

Trong phần này chúng ta chỉ xét đến việc lấy về thông tin chi tiết của email. Thiết kế table sẽ như hình dưới đây:

![Screenshot 2024-03-27 at 22 11 23](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/7076d02b-2003-4c6a-ae66-65685772ef58)

```sql
SELECT * FROM emails_by_user WHERE email_id = 123;
```

##### Query 4: Lấy toàn bộ các read hoặc unread email

Nếu domain model của chúng ta được thiết kế cho RDB, query lấy về các read email sẽ như sau:

```sql
SELECT * FROM emails_by_folder
WHERE user_id = <user_id> AND folder_id = <folder_id> AND is_read = true
ORDER BY email_id;
```

Câu query truy vấn unread email cũng hoàn toàn tương tự ngoại trừ việc đổi `is_read = false`.

Data mode cuả chúng ta được thiết kế để hướng đến NoSQL. NoSQL DB thường chỉ hỗ trợ queries trên partition và cluster keys. Do `is_read` trong bảng `emails_by_folder` không thuộc vào 2 nhóm keys nói trên nên DB sẽ reject câu query này.

Một cách làm đơn giản khác đó là lấy về toàn bộ dữ liệu trong folder, sau đó tiến hành lọc trong app. Cách làm này có thể hoạt động với các ứng dụng nhỏ nhưng với quy mô lớn, cách làm này hoạt động không hề tốt.

Vấn đề này sẽ được giải quyết bằng việc "phi chuẩn hoá" trong NoSQL. Để hỗ trợ read/ unread queries, chúng ta sẽ "phi chuẩn hoá" bảng `emails_by_folder` thành 2 bảng con:

- `read_emails`: lưu toàn bộ email ở trạng thái `read`.
- `unread_emails`: lưu toàn bộ email ở trạng thái `unread`.

Để đánh dấu một email từ `UNREAD` thành `READ`, email sẽ được xoá khỏi bảng `unread_emails` và sau đó được tạo mới trong bảng `read_emails`.

Để lấy về toàn bộ các unread emails trong một folder, ta có thể thực hiện câu query như sau:

```SQL
SELECT * FROM unread_emails
WHERE user_id = <user_id> AND folder_id = <folder_id>
ORDER BY email_id;
```

![Screenshot 2024-03-28 at 7 53 51](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/7dd2eaf2-fa37-48cd-8a19-e76ad7b256ea)

Việc phi chuẩn hoá như trên là một giải pháp khá phổ biến, có thể nó sẽ làm cho app code trở nên phức tạp và khó maintaince hơn nhưng nó sẽ giúp cải thiện hiệu năng đọc của các câu queries.

##### Bonus point: conversation threads

Threads là một tính năng được hỗ trợ bởi nhiều email clients. Nó sẽ nhóm các email replies thành một nhóm tương ứng với message gốc.

Tính năng này cho phép user có thể quét toàn bộ emails tương ứng với một cuộc hội thoại. Thông thường, thread được triển khai bằng việc sử dụng giải thuật như JWZ. Giải thuật này sẽ có logic chính như sau:

Một email header thông thường sẽ có 3 fields chính như sau:

```JSON
{
  "headers": {
    "Message-Id": "<7912d8-d1d12d12-12d12d12@gmail.com>", // message Id, được gen bởi client khi gửi message
    "In-Reply-To": "<Cad1029d1d1@gmail.com>",  // parent message Id
    "References": ["<7912d8-d1d12d12-12d12d12@gmail.com>"]  // danh sách các message Ids liên quan đến thread
  }
}
```

Với cấu trúc như trên, email client có thể "tái tạo" lại mail conversations từ các messages nếu mọi mesages trong reply chain được preloaded.

#### Đánh đổi về tính thống nhất

Một vấn đề cần phải đánh đổi phổ biến với các distributed database dựa trên replication đó là `tính thống nhất` và `tính sẵn có`.

Sự chính xác là vô cùng quan trọng với email system. Theo như thiết kế chúng ta muốn bất kì mailbox nào cũng phải có tính "duy nhất", trong trường hợp gặp lỗi (failover), mailbox không thể truy cập bởi client. Các thao tác sync/ update sẽ được dừng lại cho đến khi nào failover kết thúc.

Đây chính là việc chúng ta đánh đổi tính thống nhất về mặt dữ liệu thay cho tính sẵn có của hệ thống.

#### Email deliverability

Việc setup và gửi email không hề khó. Điều khó nhất ở đây đó là làm cho emails thực sự đến được với người nhận. Nếu emails được gửi đến spam folder thì khả năng rất cao là user sẽ không đọc nó.

Email spam là một vấn đề lớn. Trong thực tế, có hơn 50% email sẽ được gửi đến spam folder.

Nếu bạn setup một email server cho riêng mình thì khả năng cao là emails gửi từ server của bạn sẽ bị cho vào spam (do không có danh tiếng). Có những yếu tố sau là cần cân nhắc nếu muốn cải thiện `email deliverability`.

- **Dedicated IPs**. Nên có các dedicated IPs vì bản thân các email service cũng sẽ rất hạn chế việc accept các địa chỉ IP lạ, chưa từng xuất hiện trong lịch sử.
- **Classify Emails**. Gửi các loại emails khác nhau từ các địa chỉ khác nhau. Nguyên nhân là bởi nếu ta gửi cùng một loại email (marketing email) từ cùng một server thì ISPs sẽ xem đó như các emails khuyến mại, tiếp thị, ...
- **Email Sender Reputation**. Việc tận dụng các hệ thống email nổi tiếng như Office365 hay Gmail hoặc Yahoo Mail sẽ giúp mail của chúng ta ít bị cho vào spam folder hơn. Bản thân việc sử dụng Amazon Simple Email Service (SES) cũng sẽ mất 2 - 6 tuần để "làm nóng" địa chỉ IP của bạn.
- **Nhanh chóng ban các spammer**. Spammers nên bị ban sớm nhất có thể để tránh tình trạng ảnh hưởng đến email reputation.
- **Xử lí feedback**. Việc setup một feedback loop cho ISP để có thể giữ cho tỉ lệ phàn nàn (complaint rate) thấp và tỉ lệ ban các spammer account cao là rất quan trọng. Nếu email không đến được với người nhận hoặc khi user phàn nàn, một trong những outcome sau đây sẽ xảy ra.
  - `Hard bounce`. Email bị reject bởi ISP, nguyên nhân là vì địa chỉ nhận không hợp lệ.
  - `Soft bounce`. Email không đến được với người nhận vì một vài lí do mang tính chất tạm thời (VD: ISPs quá tải).
  - `Complaint`. Điều này có nghĩa là người nhận ấn vào "report spam" button.

Hình dưới đây cho thấy quá trình thu thập và xử lí bounce/ complaint. Chúng ta sẽ sử dụng các queues riêng cho `hard bounce`, `soft bounce`, `complaint`.

![Screenshot 2024-03-28 at 22 32 01](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/b071852a-5441-470c-88f4-b6e032205186)

- **Email authentication**. Để cho emails có thể hoạt động, ngoài việc có kiến thức về domain thì việc có một mối liên hệ tốt với ISPs cũng rất quan trọng.

#### Search

Thông thường việc tìm kiếm mail sẽ dựa theo các key words xuất hiện trong subject hoặc body. Ngoài ra khi email được gửi, được đọc hoặc bị xoá, ta cần tiến hành reindexing.

Email searching chỉ hoạt động khi user ấn search button. Do đó search feature sẽ thiên về write hơn là read.

Bảng dưới đây sẽ so sánh sự khác biệt giữa Google search và Email search.

|               | Scope              | Sorting                                                               | Accuracy                                                                           |
| ------------- | ------------------ | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Google Search | Toàn bộ internet   | Sắp xếp theo mức độ liên quan của các kết quả                         | Indexing thường mất thời gian nên có vài items sẽ không được hiển thị ngay lập tức |
| Email Search  | Email box của user | Sắp xếp theo các thuộc tính như thời gian, có attachment, unread, ... | Indexing thường đạt mức độ real-time và kết quả phải chính xác                     |

Để triển khai email search chúng ta có 2 cách:

- Sử dụng Elasticsearch.
- Sử dụng native search có sẵn trong data-store.

##### Option 1: Elasticsearch

High-level design cho email search sử dụng ElasticSearch sẽ giống như hình dưới đây

![Screenshot 2024-03-28 at 22 41 04](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/514f059b-d2b2-40af-9558-78f0fd4c69aa)

Do queries thường được thực hiện trên email server của user nên chúng ta có thể nhóm các documents vào cùng một node bằng cách sử dụng `user_id` như là partition key.

Khi user nhấn vào search button, user sẽ phải chờ cho đến khi có kết quả trả về, nên do đó thao tác tìm kiếm ở đây là synchronous.

Với các events như `send email`, `receive email`, `delete email` được trigger, quá trình reindexing sẽ được diễn ra và được đảm nhận bởi các offline jobs. Kafka được sử dụng ở đây như một cách để decouple các `services trigger reindexing` với các `services thực thi reindexing`.

ElasticSearch là một search-engine database rất phổ biến và nó hỗ trợ full-text search cho email rất tốt. Một vấn đề ở đây đó là việc giữ cho primary email được lưu trữ một cách đồng bộ.

##### Option 2: Custom search solution

Các hệ thống email providers lớn thường tự mình dev các search engine nhằm đáp ứng yêu cầu của bản thân hệ thống.

Việc thiết kế một search-engine không hề đơn giản. Ở đây chúng ta chỉ tập trung vào vấn đề `bottle neck` - một vấn đề mà rất nhiều search engine gặp phải.

Như trong phần `back-of-the-envelopre` dữ liệu metadata và attachment hàng ngày của chúng ta có thể lên đến cả PB, trong khi đó một email account có thể có hơn 1 triệu mail. Nên bottle-neck với index server thường nằm ở disk I/O

Quá trình xây dựng index thường khá nặng về write. Một chiến lược hay ho ở đây đó là sử dụng `Log-Structured Merge-Tree (LSM)`.

LSM Tree là core data-structure phía sau các DB như Bigtable, Cassandra.

Khi email mới đến, nó sẽ được đưa vào Level 0 (memory), khi dữ liệu ở mỗi level đạt tới ngưỡng trên, chúng sẽ được merged lại và đưa lên các level cao hơn.

Ngoài ra ta thấy rằng dữ liệu liên quan đến email thường không đổi mà chỉ có thông tin liên quan đến folder là thay đổi tuỳ theo filter rules. Do đó ta có thể chia chúng thành 2 sections, các request liên quan đến folder change sẽ được đưa vào một nhóm và chúng ta chỉ thay đổi folder chứ không thay đổi email data.

![Screenshot 2024-03-28 at 23 12 41](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/03ac9be6-4a1a-4d81-b4a3-7779a36b57a9)

Cùng so sánh giữa options nêu trên, chúng ta thấy rằng:

- **Khả năng mở rộng**. Việc custom search sẽ cho nhiều sự lựa chọn hơn.
- **Độ phức tạp hệ thống**. Với Elasticsearch, ta cần maintaince `datastore` và `Elasticsearch`. Với custom search thì chỉ có 1 system mà thôi.
- **Tính thống nhất về dữ liệu**. Với Elasticsearch do dữ liệu được lưu tại datastore và Elasticsearch nên việc đảm bảo tính thống nhất sẽ có hơn so với custom search khi chỉ có một datastore duy nhất.
- **Khả năng mất dữ liệu**. Cả 2 options đều không bị mất dữ liệu.
- **Thời gian dev**. Dễ dàng tích hợp Elasticsearch, với custom search thì sẽ phức tạp hơn.

Một kinh nghiệm trong thực tế cho thấy, các hệ thống nhỏ thường sẽ sử dụng Elasticsearch do khả năng dễ tích hợp cũng như không tốn quá nhiều thời gian dev. Với các hệ thống lớn ta cần một team dev riêng để phát triển cũng như maintain search engine của riêng mình.

Với quy mô hệ thống như Gmail, Outlook sẽ tốt hơn khi có native search bên trong DB.

#### Khả năng mở rộng và tính sẵn có

Chúng ta luôn muốn hệ thống có khả năng scale theo chiều ngang.

Để tăng tính sẵn có, dữ liệu sẽ được sao lưu lên nhiều data centers. User sẽ tương tác với email server gần họ nhất. Do quá trình phân vùng mạng (network partition), users có thể truy cập messages từ nhiều data centers khác nhau.

![Screenshot 2024-03-29 at 8 10 30](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/17dd40e8-bb13-48f2-a5fc-f561137e327d)

## Bước 4: Tổng kết

Trong chương này chúng ta đã thiết kế kiến trúc cho email servers với quy mô lớn qua các bước như:

1. Tính toán, ước chừng quy mô hệ thống.
2. High-level design: thiết kế email API cũng như phân tích việc không thể áp dụng các mail protocol truyền thống.
3. Trong phần deep-dive design: tập trung vào metadata DB, email deliverability, search, khả năng mở rộng.

Nếu có thời gian, chúng ta có thể bàn thêm về:

- Khả năng chịu lỗi (Fault tolerance): các vấn đề như node failures, network issues, event delays, ...
- Bảo mật, do bản thân email sẽ bao gồm các thông tin nhạy cảm. Gmail cung cấp một vài tính năng như safe browsing hay email encryption.
- Tối ưu hoá. Với các email có attachment, thay vì lưu trên nhiều object store (S3), ta có thể kiểm tra để chỉ cần lưu trữ ở một nơi là đủ.
