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

Write access tới read-write file cần phải được serialized. Như ở hình trên, các object được lưu trữ một cách nối tiếp nhau trong read-write file. Để đảm bảo thứ tự này, các `multiple cores` xử lí các write request song song cần phải "lần lượt" ghi vào read-write file này theo một thứ tự nhất định, với các server hiện đại với một số lượng lớn các cores, thì việc xử lí song song nhiều requests sẽ làm hạn chế đi write throughtput. Để giải quyết vấn đề này chúng ta có thể cung cấp các dedicated read-write files (1 file cho mỗi core processing các requests đến).

#### Object lookup

Mỗi một data file lưu giữ rất nhiều objects nhỏ, nhưng làm cách nào để data node có thể định vị được object bằng UUID ? Data node cần những thông tin sau:

- Data file chứa object.
- Starting offset của object trong data file.
- Kích cỡ của object.

| **object_mapping** |
| ------------------ |
| **object_id**      |
| file_name          |
| start_offset       |
| object_size        |

| **Field**    | **Description**                       |
| ------------ | ------------------------------------- |
| object_id    | UUID của object                       |
| file_name    | Tên của file chứa object              |
| start_offset | Địa chỉ bắt đầu của object trong file |
| object_size  | Số bytes trong object                 |

Với object_mapping, chúng ta sẽ xem xét 2 sự lựa chọn sau:

- File-based key-value store như RocksDB: dựa trên SSTable, `ghi nhanh nhưng đọc chậm`.
- RDB: sử dụng B+ Tree, dựa theo storage engine, `đọc nhanh nhưng ghi chậm`.

Như đã đề cập ở phần trước, pattern của chúng ta ở đây đó là `ghi một lần` và `đọc nhiều lần`. Do đó việc lựa chọn RDB sẽ tốt hơn so với RocksDB.

Làm cách nào để có thể deploy RDB này ? Ở quy mô hệ thống của chúng ta, kích cỡ của mapping table là rất lớn. Chú ý rằng mapping data là "biệt lập - isolated" hoàn toàn trong một node. Không cần thiết phải shared giữa các nodes. Để tận dụng ưu điểm này, chúng ta có thể đơn thuần là deploy một RDB trên mỗi data node. SQLite là một sự lựa chọn tốt ở đây, nó là một file-based RDB rất phổ biến.

### Flow update dữ liệu

Do chúng ta đã thay đổi ít nhiều lên data node, hãy cùng nhau xem lại cách lưu object vào data node.

1. API service gửi request để lưu một object mới với tên là `object 4`.
2. Data node service sẽ thêm object với tên là `object 4` vào cuối read-write file có tên là `/data/c`.
3. Một record mới dành cho `object 4` sẽ được thêm vào `object_mapping` table.
4. Data node service trả UUID về cho API service.

![Screenshot 2024-04-04 at 22 46 39](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ae523755-42bd-44cb-9de9-02832e3fa28b)

### Durability

Data reliablity là một yếu tố rất quan trọng đối với storage system. Làm cách nào để phát triển được một storage system đáp ứng cả những yêu cầu về độ bền (durability) của dữ liệu ? Các failure cases cần được xem xét một cách cẩn thận và dữ liệu cũng cần phải được replicated.

#### Hardware failure & failure domain

Lỗi liên quan đến phần cứng (hardware failure) là một điều không thể tránh khỏi được. Có những loại thiết bị có độ bền cao thế nhưng chúng ta không thể chỉ dựa vào một hard drive duy nhất để đảm bảo về độ bền của dữ liệu trong hệ thống. Một cách để tăng "độ bền" của dữ liệu đó là replicate dữ liệu trên nhiều hard drives, nhờ đó khi một drive gặp vấn đề thì cũng không ảnh hưởng đến tính sẵn có của dữ liệu. Trong thiết kế, chúng ta sẽ replicate dữ liệu 3 lần.

Ta giả sử rằng hard drive có tỉ lệ lỗi là 0.81%, nếu ta sao lưu dữ liệu 3 lần (tạo 3 bản sao dữ liệu) thì "độ tin cậy" sẽ là `1 - 0.0081^3 = 0.99999`.

Để đánh giá về độ bền của dữ liệu một cách hoàn chỉnh ta cũng cần xem xét đến sự ảnh hưởng của failure domain. Failure domain là một logic section hoặc một section vật lý của môi trường mà ở đó nó sẽ bị ảnh hưởng tiêu cực khi một critical service gặp vấn đề. Trong các hệ thống server lớn, các servers thường được đặt trên cùng một giá đỡ, các giá đỡ này sẽ sẽ được nhóm thành một row/ floor/ room. Trong một giá thì các servers sẽ chia sẻ chung network switches, nguồn điện, các servers trong cùng 1 giá sẽ có `rack-level failure domain`.

Một server hiện đại sẽ chia sẻ các components như `mother-board`, `processor`, `nguồn điện`, `HDD`, ... Các components trong cùng một server sẽ là `node-level failure domain`.

Trong thực tế chúng ta sẽ chia các data center infrastructure từ không share bất kì thứ gì thành phân tán trên nhiều AZ khác nhau (large-scale failure domain isolation), chúng ta sẽ replicate dữ liệu trên nhiều AZ để giảm thiểu rủi ro.

Hãy nhớ rằng, việc lựa chọn `failure domain level` không trực tiếp tăng độ bền cho dữ liệu nhưng nó sẽ giúp tăng độ tin cậy của hệ thống đặc biệt là với các hệ thống lớn khi bị mất điện hoặc gặp thiên tai, ...

![Screenshot 2024-04-04 at 22 49 51](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/0bca93a6-9a08-4255-85ab-fac90892f56f)

#### Erasure coding

Việc tạo ra 3 bản sao dữ liệu đem lại cho dữ liệu một độ bền cao hơn.

Cũng có một cách khác giúp cải thiện độ bền dữ liệu, đó là `erasure coding`. Cách tiếp cận của `erasure coding` rất khác. Nó sẽ chia nhỏ dữ liệu thành nhiều phần và đặt trên các servers khác nhau, đồng thời tạo ra các cặp "chẵn lẻ dư thừa".

Khi có lỗi xảy ra ta có thể sử dụng các dữ liệu đã được chia nhỏ và các cặp "chẵn lẻ" này để tái cấu trúc lại dữ liệu.

![Screenshot 2024-04-05 at 22 49 39](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c87418d2-d86f-4776-b3fd-472db9fde957)

1. Dữ liệu được chia thành bốn phần với kích cỡ bằng nhau (kích cỡ chẵn) d1, d2, d3, d4.
2. Công thức được sử dụng để tính toán parities p1 và p2.
3. Dữ liệu trong d3 và d4 bị mất do node crashes.
4. Sử dụng công thức để khôi phục lại dữ liệu bị mất trong d3 và d4 (bằng việc sử dụng các giá trị đã biết của d1, d2, p1 và p2).

(8+4) erasure coding sẽ chia dữ liệu thành `8 chunks` và tính `4 parities`. Tất cả `12 phần dữ liệu` này phải có cùng kích cỡ.

12 chunks data này sẽ được phân bổ lên 12 failure domains khác nhau. Công thức ở đây sẽ đảm bảo rằng, erasure encoding sẽ đảm bảo cho dữ liệu có thể được tái tạo lại khi có nhiều nhất 4 nodes bị sập.

![Screenshot 2024-04-05 at 23 15 40](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e9b515f4-2d62-4c8f-b12c-ec70e69cde09)

So sánh với replication, data router chỉ cần đọc dữ liệu của object từ một healthy node duy nhất thì trong erasure coding, data router phải đọc dữ liệu từ ít nhất 8 healthy nodes. Đây là một kiến trúc cần phải có sự đánh đổi (Architectural design tradeoff). Chúng ta áp dụng một giải pháp "phức tạp" với `tốc độ truy cập chậm hơn` nhưng đem lại `độ bền dữ liệu cao hơn` và `chi phí lưu trữ thấp hơn`. Với object storage, giá tiền phải trả chủ yếu sẽ là giá thành dùng cho việc lưu trữ thì đây là một giải pháp đáng để thử.

Một câu hỏi ở đây đó là chúng ta cần bao nhiêu `extra space` cho erasure coding? Với mỗi 2 chunks dữ liệu, chúng ta cần một parity block, do đó với 12 chunks ta cần 6 parities -> lượng bộ nhớ tăng thêm 50%. Với 3-copy replication, lượng bộ nhớ sẽ tăng 200%.

![Screenshot 2024-04-05 at 23 13 34](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c4cc4e9d-ca3d-4a48-b294-7a81ecb395e3)

Liệu rằng erasure coding có tăng độ bền của dữ liệu ? Chúng ta cùng giả sử rằng một node có 0.81% khả năng failed. Theo như công thức tính toán bởi Backblaze, erasure coding có thể đảm bảo `11 nines durability`. Tuy nhiên việc tính toán này đòi hỏi những công thức phức tạp.

Cùng so sánh ưu nhược điểm giữa replication và erasure coding.

|                   | **Replication**                                                                                                               | **Erasure coding**                                                                                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Durability        | 6 nines durability                                                                                                            | 11 nines durability **Erasure coding win**                                                                                                                       |
| Hiệu suất lưu trữ | 200% storage overhead                                                                                                         | 50% storage overhead **Erasure coding win**                                                                                                                      |
| Compute resource  | Không tính toán **Replication win**                                                                                           | Nặng về tính toán số lượng parities                                                                                                                              |
| Write performance | Replicate dữ liệu sang nhiều nodes, không cần tính toán **Replication win**                                                   | Mọi thao tác ghi đều đi liền với việc tính toán parities, từ đó làm chậm quá trình ghi dữ liệu                                                                   |
| Read performance  | Bình thường sẽ luôn đọc dữ liệu từ một replica, trong failure mode, dữ liệu được đọc từ non-fault replica **Replication win** | Bình thường, các thao tác đọc sẽ đến từ nhiều node trong cluster. Trong failure mode, dữ liệu cần được tái cấu trúc trước khi đọc nên sẽ thao tác đọc sẽ bị chậm |

Tổng kết lại, replication sẽ đảm bảo latency-sensitive application, erasure coding giúp giảm chi phí lưu trữ. Nhưng việc thiết kế sử dụng erasure sẽ phức tạp hơn so với replication, nên trong ứng dụng lần này chúng ta sẽ tập trung vào phương án replication.

### Xác nhận tính chính xác

Erasure coding giúp giảm giá thành lưu trữ và tăng độ bền cho dữ liệu. Bây giờ chúng ta sẽ chuyển sang việc giải quyết một vấn đề khó hơn đó là: "dữ liệu bị hỏng - thiếu chính xác (data corruption)".

Nếu disk fail hoàn toàn và được phát hiện, nó có thể bị coi như data node failure. Trong trường hợp này, chúng ta có thể tái cấu trúc lại dữ liệu bằng việc sử dụng erasure coding.

In-memory data corruption là một điều bình thường, hay xảy ra với các hệ thống lớn.

Vấn đề này có thể được phát hiện thông qua việc kiểm tra `checksums` giữa các boundaries của các tiến trình. Checksum là một data-block với kích thước nhỏ, được sử dụng để tìm ra data errors.

![Screenshot 2024-04-05 at 23 14 36](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/10fee526-7dcf-4c9d-88c6-8dec3faf1ba8)

Nếu chúng ta biết checksum của dữ liệu nguồn, chúng ta có thể tính checksum của dữ liệu sau khi tiến hành transmit:

- Nếu chúng khác nhau, chứng tỏ dữ liệu có vấn đề.
- Nếu trùng nhau, khả năng cao dữ liệu không gặp vấn đề gì cả. Xác suất khó có thể đạt 100% nhưng trong thực tế, chúng ta có thể giả sử rằng chúng giống nhau.

![Screenshot 2024-04-05 at 23 15 12](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/d0742a06-647f-4c6b-aed9-842a0c42223b)

Có rất nhiều loại giải thuật checksum (MD5, SHA1, HMAC), một giải thuật checksum được coi là tốt là khi output của nó có sự khác biệt rõ ràng ngay cả khi dữ liệu nguồn chỉ có một sự thay đổi rất nhỏ. Lần này chúng ta lựa chọn giải thuật MD5.

Trong thiết kế lần này, chúng ta sẽ thêm checksum vào cuối mỗi object. Trước khi file được đánh dấu là read-only, chúng ta sẽ thêm checksum của toàn bộ file vào cuối file.

![Screenshot 2024-04-05 at 23 16 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e41a5174-52da-409e-9294-fcd2ea395c7a)

Với (8 + 4) erasure coding và checksum verification, đây là những gì diễn ra khi chúng ta đọc dữ liệu:

1. Lấy về object data và checksum.
2. Tính toán checksum với dữ liệu nhận được
   a. Nếu 2 checksums trùng nhau, dữ liệu không hề gặp lỗi.
   b. Nếu checksums khác nhau, dữ liệu gặp sự cố. Chúng ta sẽ thử đọc dữ liệu từ các failure domains khác.
3. Lặp lại 2 bước 1 và 2 cho đến khi tất cả 8 phần dữ liệu được trả về, sau đó chúng ta sẽ tái cấu trúc lại dữ liệu và gửi lại nó cho client.

### Metadata data model

#### Schema

Database schema cânf hỗ trợ 3 queries như sau:sau

- Query 1: Tìm object ID bằng object name.
- Query 2: Insert và delete object dựa theo object name.
- Query 3: List các objects trong bucket có cùng prefix

![Screenshot 2024-04-06 at 17 12 09](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/877777bd-ba33-460e-83c2-89bd31ec5ae9)

Với schema này, chúng ta có 2 bảng: `bucket` và `object`.

#### Scale bucket table

Do thông thường, số lượng bucket mà 1 user tạo ra thường không nhiều nên số lượng bản ghi của `bucket table` thường sẽ nhỏ. Giả sử chúng ta có 1 triệu users, mỗi user sẽ có 10 buckets và mỗi record có kích cỡ 1KB. Do đó chúng ta cần `1 triệu x 10 x 1KB = 10GB` cho không gian nhớ.

Toàn bộ bảng có thể đưa vào một database server được nhưng một database server có thể không có đủ CPU hoặc băng thông mạng để xử lí tất cả các read requests. Do đó chúng ta sẽ chia read load trên nhiều database replicas.

#### Scale object table

Bảng `object` lưu object metadata. Dataset ở quy mô lần này sẽ không thể đưa vào một database instance duy nhất được. Ta cần scale bảng `object` bằng sharding.

Một lựa chọn để phân mảnh đó là sử dụng `bucket_id`, khi đó tất cả các objects nằm trong cùng một bucket đều được lưu trong cùng một shard. Cách làm này không ổn vì hotspot shards có thể chứa cả tỉ objects.

Một cách làm khác đó là sử dụng `object_id`, cách này sẽ giúp phân bổ object đều đặn lên các shards nhưng không thể thực hiện query 1 và 2 được do 2 câu queries này đều dựa trên URI.

Chúng ta lựa chọn giải pháp shard bằng `bucket_name` và `object_name`. Do hầu hết các thao tác liên quan đến metadata đều dựa theo object URI, ví dụ tìm object ID bằng URI hoặc upload object bằng URI. Để phân bổ dữ liệu một cách đều đặn chúng ta có thể sử dụng giá trị băm của `<bucket_name, object_name>` để làm sharding key.

Cách làm này sẽ hỗ trợ tốt cho query 1 và 2 thế nhưng lại không tốt đối với query cuối cùng.

### Liệt kê các objects trong một bucket

Object store sẽ sắp xếp các files theo cấu trúc phẳng thay vì cấu trúc kế thừa thừa (file system là cấu trúc kế thừa). Một object có thể được truy cập thông qua path với format `s3://bucket-name/object-name`. VD: `s3://mybucket/abc/d/e/f/file.txt`

- Bucket name: `mybucket`.
- Object name: `abc/d/e/f/file.txt`.

Để giúp user tổ chức các objects trong bucket, S3 đề xuất khái niệm `prefixes`. Prefix là phần xâu bắt đầu trong object name.

S3 sử dụng prefix để tổ chức dữ liệu theo cách tương tự như directories. Thế nhưng prefixes không phải là directories. Việc liệt kê dựa theo prefix sẽ chỉ cho ra kết qủa là các objects với object name bắt đầu bằng prefix.

Ở ví dụ `s3://mybucket/abc/d/e/f/file.txt` ta có `abc/d/e/f/`.

AWS S3 lisiting command có 3 cách dùng:

1. Liệt kê mọi buckets của user.

```sh
aws s3 list-buckets
```

2. Liệt kê mọi objects trong bucket với cùng level như prefix. Command sẽ như sau:

```sh
aws s3 ls s3://mybucket/abc/
```

Ví dụ với:

```txt
CA/cities/losangeles.txt
CA/cities/sanfranciso.txt
NY/cities/ny.txt
federal.txt
```

Liệt kê bucket với `/` prefix sẽ trả về kết quả như sau:

```txt
CA/
NY/
federal.txt
```

3. Liệt kê các objects trong một bucket với cùng prefix một cách đệ quy. Command trông sẽ như sau:

```sh
aws s3 ls s3://mybucket/CA/ --recursive
```

Với các objects như ở phần `2.`, kết quả thu được sẽ là:

```txt
CA/cities/losangeles.txt
CA/cities/sanfranciso.txt
```

### Single Database

Với single database, ta có thể liệt kê mọi buckets thuộc về user bằng cách chạy query như sau:

```SQL
SELECT * FROM bucket WHERE owner_id={id}
```

Để liệt kê các objects trong bucket có cùng prefix trong bucket ta có câu query:

```SQL
SELECT * FROM object
WHERE bucket_id = "123" AND object_name LIKE "abc/%";
```

### Distributed databases

Khi DB được shared, sẽ rất khó để triển khai listing function vì chúng ta không biết shard nào sẽ chứa dữ liệu. Cách làm "ổn" nhất ở đây đó là chạy search query trên tất cả các shards, sau đó "kết tập" kết quả lại. Ta có thể triển khai cách làm này như sau:

1. Metadata service sẽ queries mọi shard với câu query như sau:

```SQL
SELECT * FROM object
WHERE bucket_id = "123" AND object_name LIKE "a/b/%";
```

2. Metadata service sẽ kết tập mọi objects trả về từ các shard và trả về kết quả cho caller.

Cách làm này hoạt động nhưng việc triển khai pagination sẽ hơi phức tạp một chút. Để hiểu rằng tại sao nó lại phức tạp, hãy cùng xem pagination hoạt động như thế nào với single database. Để trả về pages với 10 objects cho mỗi page, SELECT query sẽ như sau:

```SQL
SELECT * FROM object
WHERE bucket_id = "123" AND object_name LIKE `a/b/%`;
ORDER BY object_name OFFSET 0 LIMIT 10;
```

OFFSET và LIMIT sẽ trả về 10 objects đầu tiên. Với query tiếp theo, ta sẽ set OFFSET bằng 10 để lấy về dữ liệu cho page tiếp theo.

Server sẽ trả về cursor ứng với mỗi page cho client. Thông tin về offset sẽ được mã hoá trong cursor. Client sẽ đưa thông tin về cursor vào trong request cho page tiếp theo. Server giải mã cursor và sử dụng thông tin về offset được embedded trong đó để tạo câu query cho page tiếp theo. Để tiếp tục với ví dụ phía trên, query cho page thứ hai sẽ như sau:

```SQL
SELECT * FROM metadata
WHERE bucket_id = "123" AND object_name LIKE `a/b/%`
ORDER BY object_name OFFSET 10 LIMIT 10;
```

Quá trình này cứ lặp đi lặp lại cho đến khi server trả về một cursor đặc biệt đánh dấu kết thúc quá trình listing.

Với distributed database, sẽ có những shard trả về full 10 object cho 1 page nhưng cũng có những shard chỉ trả về 1 phần hoặc có những shard không trả về kết quả nào cả.

Application code sẽ nhận kết quả trả về từ tất cả các shard, kết tập và sắp xếp sau đó sẽ trả về một page với 10 items.

Object không nằm trong page hiện thời sẽ được đưa vào các pages tiếp theo. Mỗi shard sẽ có một offset khác nhau, server cần phải theo dõi offset của các shard và kết nối chúng lại trong cursor. Nếu có cả trăm shard thì ta cần theo dõi cả trăm offset. Do object storage được điều chỉnh cho mục đích scale và tăng độ bền nên object listing performance thường hiếm khi được ưu tiên.

Trong thực tế, các commercial object storage thường hỗ trợ object listing với sub-optimal performance. Để thực hiện điều này, chúng ta có thể "phi chuẩn hoá" listing data thành một bảng riêng, được shared bằng bucketID. Bảng này chỉ dùng cho listing objects. Với thiết lập này, ngay cả khi buckets có cả tỉ objects, hệ thống vẫn sẽ có được performance ở mức chấp nhận được. Cách làm này giúp "tách biệt" listing query với single database với cách triển khai rất đơn giản.

### Object versioning

Bằng việc lưu trữ nhiều version của object, ta có thể restore objects vô tình vị xoá hoặc ghi đè. Ví dụ như việc ta có thể sửa, và lưu một document trong một bucket dưới cùng một tên, trong cùng một bucket.

Khi không có versioning thì document metadata cũ sẽ được thay thế bởi document metada mới. Version cũ sẽ được đánh dấu là `deleted`, không gian nhớ của nó sẽ bị "dọn dẹp" bởi `garbage collector`.

Khi có versioning thì tất cả các version cuả object đều được giữ lại chứ không hề bị xoá.

![Screenshot 2024-04-09 at 8 14 22](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/0c607f33-aff6-4acb-9423-b17d7afb8b33)

1. Client gửi HTTP PUT request để upload object với tên là `script.txt`.
2. API Service verify user identity và đảm bảo user có quyền `WRITE` trên bucket.
3. Sau khi đã verified, API service upload dữ liệu lên data store. Data store lưu dữ liệu dưới dạng một object mới và trả về UUID mới cho API service.
4. API service gọi metadata store để lưu thông tin metadata của object.
5. Để hỗ trợ versioning, object table cho metadata store có một cột gọi là `object_version`, cột này chỉ được sử dụng khi áp dụng versioning. Thay vì ghi đề lên record đang tồn tại, một record mới sẽ được thêm vào với cùng `object_name`, `bucket_id` nhưng khác `object_id` và `object_version`. `object_id` là UUID cho object mới được trả về từ bước 3. `object_version` là `TIMEUUID` được gen khi một bản ghi mới được đưa vào. Không quan trọng vào loại DB mà chúng ta sử dụng, điều quan trọng ở đây đó là việc tìm kiếm version hiện thời của một object. Version hiện thời sẽ có `TIMEUUID` lớn nhất trong số các bản ghi có cùng `object_name`. Hình dưới đây sẽ mô tả quá trình versioning của metadata.

Khi chúng ta xoá một object, tất cả các versions cũ vẫn tồn tại trong bucket và chúng ta sẽ insert thêm delete marker như hình dưới đây.

<img>

`Delete Marker` sẽ là version mới của object và nó sẽ trở thành current version của object. `GET` request khi tìm thấy `delete marker` sẽ trả về lỗi `404 Not Found`.

### Tối ưu hoá khi upload các files lớn

Trong phần back-of-the-envelope estimation, chúng ta giả định ra 20% số lượng objects upload lên là các objects có kích cỡ lớn (có thể hơn vài GBs), việc upload object trực tiếp là điều hoàn toàn có thể, thế nhưng sẽ có những tình huống khi kết nối mạng gặp trục trặc, thì việc upload file có thể bị đình trệ giữa chừng dẫn đến việc phải bắt đầu lại từ đầu. Một giải pháp ở đây có thể là chia nhỏ object to thành các phần nhỏ hơn, sau đó upload những phần nhỏ này lên một cách độc lập. Sau khi upload xong, data store sẽ upload các phần này thành một object hoàn chỉnh. Quá trình này được gọi là `multipart upload`.

<img>

1. Client gọi data store để khởi tạo multiplart upload.
2. Data store trả về `uploadID` cho phiên upload lần này.
3. Client chia nhỏ file to thành các phần nhỏ, giả sử kích cỡ file là 1.6GB, file sẽ được chia thành các phần nhỏ - giả sử kích cỡ mỗi phần này là 200MB, do đó file sẽ được chia thành 8 phần. Phần đầu tiên sẽ được upload cùng với `uploadID` nhận được ở bước 2.
4. Sau khi một phần của file được upload xong, data store sẽ trả về `ETag` - md5 checksum của phần đó, nó được sử dụng để verify multipart upload.
5. Sau khi tất cả các phần đã được upload xong, client sẽ gửi complete multipart upload request với (uploadId, Part number và ETag tương ứng)
6. Data store sẽ lặp ghép các phần object được upload dựa theo part number. Do object có kích cỡ lớn nên quá trình lắp ghép này sẽ tốn thời gian, sau khi có được object hoàn chỉnh, success message sẽ được trả về cho client.

Một vấn đề tiềm tàng khác có thể phát sinh ở đây đó là sau khi tiến hành lắp ghép và thu được object hoàn chỉnh xong, các phần cũ được upload trước đó sẽ trở nên vô nghĩa nên ta cần có một cơ chế dọn dẹp các phần dữ liệu thừa này. Và đây chính là công việc của `garbage collection service`.

### Garbage collection

Garbage collection là quá trình tự động dọn dẹp các storage space không dùng đến nữa. Có một vài cách để dữ liệu trở thành "garbage":

- Lazy object deletion, object được đánh dấu là `delete marked` nhưng không hề được xoá.
- Orphan data. Dữ liệu được upload một nửa hoặc bị bỏ dở giữa chừng trong quá trình multipart upload.
- Dữ liệu không pass verify checksum.

Garbage collection sẽ không xoá dữ liệu khỏi data store mà thay vào đó deleted object sẽ được dọn dẹp định kì dựa theo cơ chế nén - compaction mechanism.

Garbage collection cũng có trách nhiệm phải dọn dẹp dữ liệu ở tất cả các replicas. Với replication, chúng ta sẽ xoá object ở cả primary lẫn backup node.

Hình dưới đây sẽ mô tả về cơ chế compaction hoạt động.

<img>

1. Garbage collector sẽ copy các objects từ file `/data/b` sang file `/data/d` ngoại trừ object 2 và object 5 vì chúng được đánh dấu là sẽ bị xoá.
2. Sau khi copy objects sang file mới, `object_mapping_table` sẽ được cập nhật. `obj_id` vẫn sẽ như cũ chỉ có `file_name` và `offset` là thay đổi ứng với việc re-location cho object ở file mới. Để đảm bảo tính thống nhất về mặt dữ liệu, `file_name` và `offset` sẽ được đưa vào transaction.

Để tránh việc tạo ra quá nhiều file mới, garbage collector sẽ chờ cho đến khi có một số lượng read-only files đủ lớn thì mới compact để số lượng các files mới tạo ra là ít nhất.

## Bước 4: Tổng kết

Phần thiết kế này tập trung vào thiết kế object storage, chúng ta đã liệt kê:

- Cách upload
- Cách download
- Cách liệt kê các object
- Object versioning

Object storage là tổ hợp của data store và metadata store. Chúng ta cũng đã nói về việc dữ liệu sẽ được lưu trữ như thế nào trong data store cũng như cách thức tăng tính tin cậy và độ bền của dữ liệu: replication và ensure coding. Với metadata store chúng ta cũng đã nói về việc multipart upload hoạt động như thế nào và cách thức thiết kế database schema để hỗ trợ các use-cases điển hình.

Đồng thời việc sharding metadata store cũng sẽ hỗ trợ các data volume lớn hơn.
