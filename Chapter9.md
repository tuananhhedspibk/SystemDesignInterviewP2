# Chương 9: Thiết kế storage system như S3

## Storage system 101

Ở high-level, storage system có 3 loại chính như sau:

- Block storage
- File storage
- Object storage

### Block storage

Trong thực tế đó là HDD, SSD.

Block storage cung cấp cho server các raw block dưới hình thức volume. Block storage có thể format raw block và sử dụng dưới dạng file system

### File storage

Nằm ở tầng phía trên của blog storage. Nó cung cấp một abstraction level cao hơn để dễ dàng xử lí files và folders. Dữ liệu được lưu dưới dạng file trong cây kế thừa thư mục (folders), đây là một giải pháp lưu trữ khá phổ biến.

File storage có thể được truy cập bởi nhiều servers cùng một lúc thông qua các file-level protocol như `SMB/CFS` hay `NFS`. Server kết nối tới file storage không cần thiết phải quan tâm đến việc quản lí blocks phía dưới hay format volume. Sự đơn giản hoá mà file storage cung cấp là một giải pháp hữu hiệu cho việc lưu trữ các sharing files hay folders với số lượng lớn.

### Object storage

Đây là một giải pháp mới, thế nhưng nó đòi hỏi việc đánh đổi giữa hiệu năng với các yêu tố như độ bền, giá thành. Giải pháp này nhắm tới các "cold" data cho mục đích archival hay backup.

Object storage lưu dữ liệu dưới dạng object theo cấu trúc "flat". Việc truy cập đến object thường thông qua RESTful API. Nó thường chậm hơn khi so với các storage types khác. Các giải pháp phổ biến hiện nay đó là AWS S3 hay Google object storage.

### So sánh

![Screenshot 2024-03-29 at 22 27 53](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/b3c4e204-2355-43eb-8cd7-cb0c2ebc29ad)

|                 | Block storage                                           | File storage                    | Object storage                                             |
| --------------- | ------------------------------------------------------- | ------------------------------- | ---------------------------------------------------------- |
| Mutable content | Y                                                       | Y                               | N (hỗ trợ object versioning, không hỗ trợ update in-place) |
| Giá             | Cao                                                     | Vừa -> Cao                      | Thấp                                                       |
| Hiệu năng       | Cao                                                     | Cao                             | Vừa                                                        |
| Tính thống nhất | Mạnh                                                    | Mạnh                            | Mạnh                                                       |
| Thích hợp với   | Virtual Machines, ứng dụng yêu cầu hiệu năng cao như DB | File system access thông thường | Binary Data, unstructured data                             |

### Các thuật ngữ

Để thiết kế một hệ thống lưu trữ tương tự như S3, ta cần hiểu được các thuật ngữ quan trọng trong một hệ thống lưu trữ.

- **Bucket**. Logical container dùng để lưu các objects, tên của nó là **global unique**, để upload dữ liệu lên S3 đầu tiên ta cần tạo bucket.
- **Object**. Là một phần dữ liệu lưu trữ trong bucket. Nó bao gồm object data (payload) và metadata (name-value pairs mô tả object).
- **Versioning**. Tính năng hỗ trợ lưu trữ nhiều phiên bản của object trong cùng bucket. Thường được enabled ở bucket-level. Tính năng này cho phép chúng ta có thể recover objects đã bị xoá hoặc bị ghi đè không như ý muốn.
- **URI**
- **Service-level agreement (SLA)**. Như một cam kết giữa service provider với client. Ví dụ Amazon S3 cung cấp SLA như sau:
  - Độ bền: 99.9999999% (Multiple AZ).
  - Tính sẵn có: 99.9%

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Yêu cầu về chức năng của hệ thống:

- Hỗ trợ các tính năng: Tạo bucket, upload & download object, object versioning, liệt kê các objects trong một bucket.
- Lưu trữ các object lớn (dung lượng lên đến vài GBs) và một số lượng lớn các object nhỏ (dung lượng vài chục KBs).
- Lượng dữ liệu lưu trữ trong 1 năm là 100PB.
- Data durability: 99.9999%, Service availability: 99.99%.

Yêu cầu "phi tính năng":

- 100PB data.
- Độ bền dữ liệu là 6 số chín (99.9999%).
- Tính sẵn có của service là 4 số chín (99.99%).
- Giảm chi phí lưu trữ cũng như tối ưu hoá về hiệu năng và độ tin cậy.

### Back-of-the-envelope estimation

Object storage thường có bottlenecks hoặc là do dung lượng đĩa hoặc do disk IO trên s (IOPS).

**Disk capacity**. Cùng giả sử các objects sẽ phân bổ như sau:

- 20% là object nhỏ (< 1MB).
- 60% là medium size (1MB ~ 64MB).
- 20% là object lớn (> 64MB).

**IOPS**. Giả sử một ổ cứng có khả năng thực hiện 100 ~ 150 bước nhảy trên giây (100 ~ 150 IOPS).

Với các giả thiết như trên, ta có thể ước chừng được tổng số objects mà hệ thống có thể lưu trữ. Để đơn giản phép tính, ta sẽ coi kích cỡ từng loại object như sau (0.5MB cho object nhỏ, 32MB cho medium size, 200MB cho object lớn). Với tỉ lệ 40% bộ nhớ được sử dụng ta có:

```txt
100PB = 10^11 MB
10^11 * 0.4 / (9,2 * 0.5MB + 0.6 * 32MB + 0.2 * 200MB) = 0.68 tỉ objects
```

Nếu ta giả sử metadata cho mỗi object có kích cỡ là 1KB, ta cần 0.68TB để lưu trữ metadata.

## Bước 2: High level design

Trước khi đi vào phần thiết kế, chúng ta hãy cùng xem xét một vài thuộc tính khá thú vị về object storage.

**Tính bất biến của object**. Object lưu trữ trong object storage là bất biến. Chúng ta có thể xoá hoặc thay thế chúng bằng một version mới chứ không thể thay đổi trực tiếp object được.

**Key-value store**. Chúng ta có thể sử dụng URI để lấy về object data. Với object URI là key, object data là value.

**Ghi một lần, đọc nhiều lần**. Data access pattern cho object data là ghi một lần và đọc nhiều lần. Theo một nghiên cứu của Linkedin, 95% request là thao tác đọc.

**Hỗ trợ cả objects nhỏ và lớn**. Kích cỡ của object rất đa dạng, chúng ta cần hỗ trợ cả hai.

Nguyên tắc thiết kế của object storage khá giống với UNIX file system. Trong UNIX, chúng ta lưu file trong local file system, filename và file-data không được lưu cùng nhau. Thay vào đó filename được lưu vào một cấu trúc gọi là "inode", file-data được lưu ở một phân vùng khác của ổ đĩa. Inode bao gồm list các file block pointers, các pointers này sẽ trỏ đến các phân vùng trên đĩa chứa file-data tương ứng.

Khi đọc file:

1. Fetch metadata từ inode
2. Đọc file-data trong phân vùng mà con trỏ tại inode trỏ tới.

Với object storage, ta cũng có cơ chế tương tự, ở đây:

- `inode` → `metadata store`.
- `hard disk` → `data store`.

Trong UNIX, inode sử dụng `file block pointer` để ghi nhận vị trí của dữ liệu trên ổ đĩa.

Trong object storage, `metadata store` sử dụng `ID` của object đển tìm đến object data trong data store qua network request.

Hình dưới đây sẽ minh hoạ cho hai cơ chế trên.

![Screenshot 2024-04-01 at 8 09 04](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/160e617c-b28f-40fe-87f0-2a183f601ac8)

Việc chia metadata và object data thiết kế trở nên đơn giản hơn. Cũng như có thể triển khai hoặc tối ưu hoá 2 components này một các độc lập.

- DataStore bao gồm các **immutable data**.
- MetaData Store bao gồm các **mutable data**.

Hình dưới đây sẽ cho thấy bucket và object trông như thế nào.

![Screenshot 2024-04-01 at 22 45 43](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/63bf9f58-5084-4d56-821a-244ef04b4b56)

### High-level design

![Screenshot 2024-04-01 at 22 50 33](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/62d2edc3-acd3-441b-b710-74e1f7266a5d)

**Load balancer**. Phân bổ đều các requests đến các API servers.

**API service**. Lần lượt gọi đến `Identity & Access Management`, `Metadata Store`, `Data Store`. Service này là stateless nên hoàn toàn có thể thực hiện **horizontal scale** với nó.

**Identity & access management (IAM)**. Phần tử trung tâm trong hệ thống tiến hành xử lí authentication, authorization, access control.

**Data Store**. Lưu và truy xuất object data. Mọi thao tác liên quan đến dữ liệu đều dựa trên object ID (UUID).

**Metadata store**. Lưu metadata của objects.

Lưu ý rằng hai khái niệm `metadata` và `data store` chỉ là các khái niệm mang tính chất logic.

Sau đây sẽ là các workflow quan trọng đối với một object storage.

- Upload object
- Download object
- Object versioning và listing objects trong một bucket.

#### Upload object

![Screenshot 2024-04-01 at 22 53 39](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/63e3d10d-2b44-425d-a0fb-430faf1fbdb7)

Object bắt buộc phải nằm bên trong một bucket. Trong ví dụ ở hình trên, đầu tiên chúng ta sẽ tạo một bucket có tên là `bucket-to-share`, sau đó tiến hành upload một file có tên là `script.txt` lên bucket.

1. Client gửi HTTP PUT request để tạo bucket với ên `bucket-to-share`, request sẽ được forward đến cho API service.
2. API service sẽ gọi đến IAM để đảm bảo user đã được xác thực và có quyền WRITE.
3. API service sẽ gọi metadata store để tạo entry với bucket info, lưu thông tin đó vào metadata DB. Khi entry được tạo thành công, success message sẽ được trả về phía client.
4. Sau khi bucket được tạo, client gửi HTTP PUT request để tạo object với tên là `script.txt`.
5. API service verify user và đảm bảo user có quyền WRITE lên bucket.
6. Sau khi quá trình validation kết thúc, API service sẽ gửi object data có trong request payload lên data store. Data store sẽ lưu payload như một object và trả về UUID của object.
7. API service sẽ gọi metadata store để tạo entry mới trong metadata DB. Nó bao gồm các thông tin quan trọng như `object_id`, `bucket_id`, `ọbect_name`, ...

#### Download object

<img>
