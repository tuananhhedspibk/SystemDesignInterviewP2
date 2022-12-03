# Chuơng 1: Proximity service

Proximity service được sủ dụng để tìm các địa điểm gần với vị trí của người dùng như nhà hàng, ga tàu, ...

Đây là core component của tính năng tìm k nhà ga gần nhất của google maps.

Trong phần này ta sẽ thiết kế một hệ thống tìm kiếm các nhà hàng, khách sạn, ... gần người dùng nhất

## Bước 1: Hiểu vấn đề và định hình design scope

Sẽ rất khó để thiết kế đầy đủ mọi chức năng trong thời gian phỏng vấn ngắn ngủi, do đó ta chỉ tập trung vào tính năng chính đó là tìm kiếm dựa theo vị trí người dùng.

Dưới đây là một vài câu hỏi nên được hỏi khi phỏng vấn.

- Liệu có cho phép search theo bán kính trong trường hợp có quá ít các businesses xung quanh (Có nếu đủ thời gian, tuy nhiên ở đây ta chỉ tập trung vào businesses trong một phạm vi bán kính cụ thể)
- Bán kính tối đa sẽ là 20km
- Người dùng có thể thay đổi bán kính tìm kiếm trên UI (Có, sẽ có các mức 5km, 10km, ...)
- Các businesses có thể cập nhật thông tin của mình ? (Có, tuy nhiên ta giả định mọi cập nhật, thay đổi sẽ được phản ánh lên hệ thống vào ngày hôm sau)
- Khi user di chuyển liệu có cần refresh page để cập nhật khoảng cách ? (Không cần, ta giả sử bước đi của người dùng là rất nhỏ)

**Các tính năng cần có:**

- Trả về toàn bộ các businesses dựa theo bán kính tìm kiếm và user location (kinh độ và vĩ độ).
- Businesses có thể thêm mới, cập nhật, xoá thông tin của mình, tuy nhiên các thông tin này không cần phải phản ánh real-time lên hệ thống.
- User có thể xem thông tin chi tiết về business.

**Các yêu cầu khác:**

- Low latency.
- Data privacy: hệ thống sử dụng thông tin về vị trí của người dùng rất nhiều do đó cần phải tuân thủ các quy định về privacy.
- High availability và scalability: hệ thống có thể chịu tải vào các giờ cao điểm hoặc các khu vực sầm uất, tấp nập.

**Back-of-the-envelopre estimation:**

Giả sử chúng ta có 100 triệu DAU và 200 triệu businesses

Một ngày sẽ có **24 * 60 * 60 = 86,400 ~ 10^5 (s)**
Một user sẽ có 5 search queries 1 ngày
Search QPS = **100 triệu * 5 / 10^5 = 5000**

## Bước 2: High-level design

### API Design

Sử dụng Restfult API.

#### GET /v1/search/nearby

Endpoint này sẽ trả về các businesses dựa theo tham số tìm kiếm. Kết quả sẽ thường được paginate nhưng trong chương này ta sẽ không đề cập tới nó.

Params:

- latitude: vĩ độ - **decimal**
- longtitude: kinh độ - **decimal**
- radius: bán kính tìm kiếm - **int**

Với thông tin về business ta cần các Endpoints riêng để lấy về những thông tin này.

#### APIs cho business

- GET /v1/businesses/:id (lấy thông tin về business)
- POST /v1/businesses (tạo mới business)
- PUT /v1/businesses/:id (cập nhật thông tin business)
- DELETE /v1/businesses/:id (xoá business)

### Data model

#### Read/ Write ratio

Tỉ lệ đọc sẽ cao hơn do:

- Người dùng đa phần sẽ tìm kiếm các businesses xung quanh mình
- Người dùng cũng cần xem các thông tin chi tiết về business

> Với các hệ thống đọc nhiều thì relational database như MySQL là một sự lựa chọn tốt

#### Data schema

Key tables:

- Business table
- Geo index table

##### Business table

Gồm thông tin chi tiết về business

![IMG_0892](https://user-images.githubusercontent.com/15076665/205073484-5690fa0b-5d15-4a91-879f-7084311d79f3.jpg)

### High-level design

Hệ thống bao gồm 2 phần chính:

- Location-based service (LBS)
- Business-related service

![IMG_0892](https://user-images.githubusercontent.com/15076665/205076892-bf4c9e7e-cbc7-4886-bd4f-c8b92126f9d9.png)

#### Location-based service (LBS)

LBS là phần quan trọng nhất của hệ thống, nó đảm nhận nhiệm vụ tìm kiếm các business gần nhất dựa theo `vị trí` và `bán kính` của người dùng.

Service này:

- Hoàn toàn chỉ đọc chứ không ghi
- QPS cao, đặc biệt trong giờ cao điểm hoặc khu vực sầm uất
- Stateless nên dễ dàng scale horizontal

#### Business service

Gồm 2 nhiệm vụ chính:

- Business owner có thể tạo mới, cập nhật, xoá business. Chủ yếu là thao tác ghi, tuy nhiên QPS thấp.
- User có thể xem thông tin về business. Chỉ đọc chứ không ghi, tuy nhiên QPS khá cao vào giờ cao điểm.

#### Database Cluster

Được triển khai theo hướng `primary-secondary` trong đó:

- Primary database sẽ dùng để ghi
- Các secondary (replicas) sẽ chuyên dùng để đọc và được đồng bộ hoá từ phía primary database

Có một vấn đề ở đây đó là quá trình replicate dữ liệu từ `primary database` sang `secondary database` sẽ làm ảnh hưởng đến việc truy xuất thông tin về business của người dùng tuy nhiên các thông tin về business cũng không bị thay đổi nhiều và cũng không yêu cầu phải được cập nhật real-time.

#### Khả năng scale của business service và LBS

Do business service và LBS đều là `stateless server` do đó rất dễ dàng để:

- Scale up trong khung giờ cao điểm
- Scale down trong sleep time

Nếu triển khai trên cloud service thì việc triển khai service ra các region hoặc zones khác nhau cũng không phải là vấn đề quá khó khăn ở đây.

### Giải thuật để lấy về các businesses gần người dùng nhất

Các dữ liệu về không gian địa lý, toạ độ hiện có sẵn với các phiên bản Redis hoặc Postgres nhưng điều quan trọng khi tiến hành phỏng vấn đó là việc ta cần hiểu được cơ chế hoạt động của `geospatial index` hơn là chỉ đơn thuần đưa ra tên của DB.

#### Giải thuật 1: Tìm kiếm 2 chiều

Giải thuật này khá đơn giản, ta chỉ thuần tuý vẽ một đường tròn với bán kính cho trước, sau đó quét các business trong phạm vi của đường tròn đó.

![Screen Shot 2022-12-03 at 12 58 22](https://user-images.githubusercontent.com/15076665/205421771-f1c8eabf-1c60-456b-b50c-8ed24e5ca22a.png)

Quá trình này có thể được mô tả bằng mã SQL giả sau:

```SQL
SELECT business_id, latitude, longtitude FROM businesses
WHERE (latitude <= {:my_lat} + radius AND latitude >= {:my_lat} - radius)
  AND (longtitude <= {:my_long} + radius AND longtitude >= {:my_long} - radius)
```

Câu query này khá đơn giản tuy nhiên nó lại gặp vấn đề về hiệu năng, nguyên nhân là do nó phải quét qua **toàn bộ dữ liệu trong bảng** dẫn đến việc tốc độ truy vấn bị ảnh hưởng rất nhiều. Ngay cả khi ta đánh index cho 2 cột `latitude` và `longtitude` thì hiệu năng cũng không được cải thiện nhiều lắm. Cốt lõi ở đây đó là việc ta cần "join" 2 datasets lại (đây là thao tác **CỰC KÌ TỐN KÉM CHI PHÍ CŨNG NHƯ THỜI GIAN**) như hình minh hoạ bên dưới.

![IMG_0894](https://user-images.githubusercontent.com/15076665/205422422-6343a697-520c-49ce-85c8-fd95fe48ce2d.jpg)

Việc đánh index chỉ hiệu quả khi dữ liệu là **dữ liệu một chiều**. Tuy nhiên ta vẫn có thể map dữ liệu 2 chiều thành 1 chiều.

Trước khi đi sâu vào việc mapping dữ liệu 2 chiều thành 1 chiều, ta sẽ nói về **các phương thức đánh index**

![IMG_0895](https://user-images.githubusercontent.com/15076665/205422881-004c3738-2044-4900-a20c-a7d812124fa9.jpg)

Trong phần này ta chỉ tập trung vào các phương thức:

- Hash: even grid, geohash
- Tree: Quadtree, Google S2

do chúng đều là các phương thức phổ biến nhất hiện nay.

Cách thức triển khai các thuật toán có thể khác nhau tuy nhiên tư tưởng chung đều là **Chia nhỏ map thành các khu vực con, đánh index từng phần để tiến hành tìm kiếm nhanh hơn**

Tiếp theo ta sẽ nói về các giải thuật trên.

#### Giải thuật 2: Evenly divided grid

Cách làm ở đây đó là chia bản đồ thế giới thành dạng lưới, túm các businesses lại theo từng lưới khác nhau và một business chỉ thuộc về 1 lưới duy nhất mà thôi.

![IMG_0896](https://user-images.githubusercontent.com/15076665/205422998-9ecb5893-94f8-4f45-aad1-3db2a33f8f2d.jpg)

Cách làm này có một vấn đề đó là việc các businesses trên thế giới phân bổ không đều đặn (các khu vực thành phố lớn như NewYork sẽ có mật độ business cao hơn so với các đại dương) từ đó dẫn đến sự phân bổ dữ liệu không cần bằng giữa các lưới. Điều ta muốn ở đây là một hệ thống lưới dày hơn ở các khu vực sầm uất và một hệ thống lưới mỏng hơn ở các khu vực "ít sầm uất".

#### Giải thuật 3: Geohash

Giải thuật này sẽ chuyển từ toạ độ 2 chiều sang một dãy số 1 chiều, giải thuật này sẽ chia bản đồ thế giới thành các ô vuông con một cách đệ quy (các ô con này sẽ ngày một nhỏ đi nếu bị chia càng nhiều và càng sâu)

![IMG_0897](https://user-images.githubusercontent.com/15076665/205423308-8f71fc23-63e7-4aa8-a47c-8c71648712fd.jpg)

Đầu tiên ta chia bản đồ thế giới thành 4 mảnh như hình trên

- `Vĩ độ` thuộc khoảng `[-180, 0]` sẽ là `0`
- `Vĩ độ` thuộc khoảng `[0, 180]` sẽ là `1`
- `Kinh độ` thuộc khoảng `[-90, 0]` sẽ là `0`
- `Kinh độ` thuộc khoảng `[0, 90]` sẽ là `1`

Sau đó với từng mảnh con ta lại chia nhỏ thành **4 mảnh con nhỏ hơn nữa**

![IMG_0898](https://user-images.githubusercontent.com/15076665/205423998-a00b9763-3ca4-46e2-b793-771fd56b143d.jpg)

Cứ thế ta chia nhỏ dần cho đến khi đạt đến size mong muốn. Geohash thường sử dụng `base32` để biểu diễn dãy "toạ độ" thu được.
Ví dụ:

1001 10110 01001 10000 11011 11010 -> `9q9hvu` (base32)

Geohash có 12 mức, tương ứng với mỗi mức là một kích thước "mảnh" khác nhau. Ta thường chỉ quan tâm đến mức từ 4 -> 6, nguyên nhân là do các mức dưới 4 thường sẽ quá to còn các mức trên 6 thì lại quá nhỏ.
