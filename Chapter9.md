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

<img>

<img>

### Các thuật ngữ

Để thiết kế một hệ thống lưu trữ tương tự như S3, ta cần hiểu được các thuật ngữ quan trọng trong một hệ thống lưu trữ.

- **Bucket**. Logical container dùng để lưu các objects, tên của nó là **global unique**, để upload dữ liệu lên S3 đầu tiên ta cần tạo bucket.
- **Object**
