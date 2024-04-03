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

Bucket không có cấu trúc kế thừa thế nhưng chúng ta có thể tạo một cấu trúc kế thừa logic cho nó bằng việc nối "bucket name" và "object name" để mô tả cấu trúc folder. VD: chúng ta sẽ chỉ định tên object là `bucket-to-share/script.txt` thay vì `script.txt`. Để lấy về một object, chúng ta sẽ chỉ định object name trong GET request. API download object sẽ như sau:

![Screenshot 2024-04-02 at 7 55 16](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/b266c1c5-cd68-47df-aefe-b65947ac2846)

Như đã nói ở phần trước, data store không lưu object nam mà sẽ hỗ trợ các thao tác thông qua `object_id (UUID)`. Do đó để lấy về object, đầu tiên ta cần map object name với UUID. Workflow sẽ như sau:

1. Client gửi HTTP GET request tới load balancer: `GET /bucket-to-share/script.txt`.
2. API service sẽ queries IAM để xác nhận rằng user có quyền `READ` với bucket.
3. Sau khi validated, API service fetch object UUID từ metadata store.
4. Tiếp theo API service sẽ lấy về object từ data store thông qua UUID.
5. API service trả về object cho client trong GET response.

## Bước 3: Design Deep Dive

### Data store

API service nhận các external requests từ user và gọi đến các internal services để hoàn thiện requests đó. Để lưu hoặc lấy về object, API service gọi data store. Hình dưới đây minh hoạ cho quá trình tương tác giữa API service và data store để upload hoặc download object

![Screenshot 2024-04-03 at 7 59 48](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ab2c2c8e-3939-43e3-8a44-62f362bc6420)

#### High-level design cho data store

Các main components của data store sẽ như sau

![Screenshot 2024-04-03 at 8 07 10](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/259d1391-1f73-4d16-b52a-416730f1bd30)

#### Data routing service

Data routing service cung cấp RESTful hoặc gRPC APIs để truy cập đến các node cluster. Do là stateless service nên nó có thể được scaled bằng cách thêm các servers mới. Service này có các nhiệm vụ như sau:

- Query đến `placement service` để lấy về data node tốt nhất cho mục đích lưu trữ dữ liệu.
- Đọc dữ liệu từ data nodes và trả về cho API service.
- Ghi dữ liệu lên data node.

#### Placement Service

Sẽ chỉ định cụ thể data nodes nào (primary hay replicas) sẽ được chọn để lưu objects. Nó bao gồm một `virtual cluster map`, map này sẽ cung cấp mô hình topo vật lý của cluster. Virtual cluster map sẽ bao gồm thông tin về vị trí của các data nodes - các thông tin này sẽ được placement service sử dụng để đảm bảo rằng các replicas được thực sự phân chia.

![Screenshot 2024-04-03 at 22 23 52](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/756e78d0-eb8b-4e71-9800-e60b6b6f9239)

Placement service liên tục monitor các data nodes thông qua heart-beats. Nếu một data node không gửi tín hiệu heart-beat trong khoảng thời gian được quy định trước (có thể là 15s) thì placement service sẽ coi như node đó bị "sập".

Đây là một service rất quan trọng do đó nên build một cluster với 5 hoăc 7 placement service node với Paxos hoặc Raft consensus protocol.

Consensus protocol đảm bảo rằng chừng nào có hơn một nửa số nodes là "healthy" thì toàn bộ hệ thống vẫn coi như đang hoạt động tốt.

Lấy ví dụ với placement service cluster với 7 nodes, khi đó cluster này có thể chịu tối đa 3 nodes failure.

#### Data node

Là nơi lưu trữ dữ liệu thực sự của object. Nó cần đảm bảo:

- Tính tin cậy.
- Độ bền.

bằng việc replicate dữ liệu trên nhiều data nodes (còn được gọi là `replication group`).

Mỗi data node sẽ có data service daemon chạy trên nó. Data service daemon sẽ liên tục gửi heart-beat đến placement service. Heart-beat message sẽ bao gồm các thông tin cơ bản như sau:

- Có bao nhiêu disk drive (HDD hoặc SSD) mà data node quản lí ?
- Lượng dữ liệu lưu trên mỗi drive là bao nhiêu ?

Khi placement service nhận heartbeat lần đầu tiên, nó sẽ gán ID cho mỗi data node và thêm nó vào virtual cluster map và trả về các thông tin như sau:

- Unique ID của data node
- Virtual cluster map
- Nơi replicate dữ liệu

#### Flow lưu trữ dữ liệu

![Screenshot 2024-04-03 at 22 36 50](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/27086daf-0162-4b69-9714-09a967de2bd0)

1. API service sẽ gửi object data tới cho `data store`.
2. Data routing service sẽ gen UUID cho object và queries placement service để tìm data node lưu object. Placement service sẽ check `Virtual cluster map` và trả về `primary data node`.
3. Data routing service sẽ gửi dữ liệu trực tiếp tới `primary node` với UUID.
4. Primary data node lưu dữ liệu trong local của mình, sau đó tiến hành sao lưu sang 2 replicate node còn lại. Primary node response lại phía data routing service khi dữ liệu được sao lưu hoàn toàn sang các secondary nodes còn lại.
5. UUID của object (ObjId) được trả về cho API service.

Ở bước 2, input sẽ là UUID của object, placement service sẽ tìm replication group cho object. Làm cách nào placement service có thể thực hiện được điều đó ? Hãy nhớ rằng việc tìm kiếm này sẽ mang tính quyết định và rất quan trọng không những thế nó phải xử lí được ngay cả khi có replication groups mới được thêm vào hoặc bị xoá đi.

Consisten hashing là một giải pháp chung phổ biến cho những chức năng tìm kiếm kiểu này.

Tại bước 4, trước khi response lại cho data routing service, dữ liệu phải được sao lưu thành công sang tất cả các secondary services, việc này đảm bảo tính thống nhất "mạnh" - strong consistency của dữ liệu nhưng cái giá phải trả đó là độ trễ khi chúng ta phải chờ cho đến khi quá trình replicate trên secondary node chậm nhất kết thúc thì mới thôi.

![IMG_2830](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/bdd66ee6-cca8-47a0-8d70-326eb2892a31)

Hình trên đây cho thấy sự đánh đổi (trade-off) giữa tính thống nhất (consistency) và độ trễ (latency).

1. Dữ liệu được xem là lưu thành công chỉ khi nó được lưu trên tất cả các node. Cách tiếp cận này đảm bảo tính thống nhất cao nhưng độ trễ lớn.
2. Dữ liệu được xem là lưu thành công khi nó được lưu thành công trên primary và 1 trong số các secondary nodes. Cách tiếp cận này đảm bảo tính thống nhất về mặt dữ liệu cũng như độ trễ ở mức trung bình.
3. Dữ liệu được xem là lưu thành công khi nó chỉ cần được lưu thành công trên primary mà thôi, cách tiếp cận này đảm bảo tính thống nhất kém nhất nhưng lại cho độ trễ thấp nhất.

Cả 2 cách tiếp cận 2 và 3 đều được xem là `eventual consistency`.

#### Dữ liệu được tổ chức như thế nào

Cách xử lí đơn giản nhất ở đây đó là lưu từng object lên từng file riêng. Cách làm này "chạy được" nhưng phải "trả giá" khi có rất nhiều file nhỏ. Hai cái giá phải trả ở đây đó là:

1. Lãng phí nhiều data blocks. File system sẽ lưu files trên các disk block rời rạc, mỗi một block sẽ có kích cỡ trung bình là 4KB, việc lưu những file nhỏ (<4KB) sẽ làm lãng phí block khi không tận dụng được hết không gian nhớ của một block đồng thời việc lưu vào block (dù lưu không hết không gian nhớ) cũng sẽ làm cho block đó coi như bị chiếm dụng hoàn toàn.
2. Có thể gây ra việc tràn số lượng các inode của hệ thống. File system lưu location cũng như các thông tin khác về một file trong một block đặc biệt gọi là inode. Hầu hết các file system đều thiết lập một số lượng các inode cố định khi tiến hành khởi tạo hệ thống. Với hàng triệu file nhỏ, tất cả các inodes của hệ thống sẽ bị chiếm dụng hết. Hơn nữa OS xử lí số lượng lớn các inodes không hề tốt.

Vì 2 lí do trên, việc lưu object dưới dạng file trên thực tế là một sự lựa chọn rất tồi.

Chúng ta có thể giải quyết vấn đề bằng việc merge các objects nhỏ vào một file lớn hơn. Về cơ bản nó sẽ hoạt động giống như write-ahead log (WAL). Khi chúng ta lưu một object, nó sẽ được thêm vào một read-write file đã có sẵn. Khi dung lượng của file đạt tới một ngưỡng đã thiết lập trước đó (thường là vài GBs), `read-write file` sẽ trở thành `read-only` và `read-write file` mới được tạo ra để nhận về các object mới. Khi file trở thành `read-only`, nó sẽ chỉ đáp ứng được các `read-request` mà thôi.

Hình dưới đây sẽ mô tả quá trình trên.

![Screenshot 2024-04-03 at 22 45 34](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/d6878592-9722-4a82-ade3-eb10afd186d6)

Write access tới read-write file cần phải được serialized. Như ở hình trên, các object được lưu trữ một cách nối tiếp nhau trong read-write file. Để đảm bảo thứ tự này, `multiple cores` xử lí các w
