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
