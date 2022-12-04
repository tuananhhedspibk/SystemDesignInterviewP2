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

**Business table:**

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

Geohash có 12 mức (length), tương ứng với mỗi mức là một kích thước "mảnh" khác nhau. Ta thường chỉ quan tâm đến mức từ 4 -> 6, nguyên nhân là do các mức dưới 4 thường sẽ quá to còn các mức trên 6 thì lại quá nhỏ.

Vậy làm cách nào để có thể chọn được mức geohash phù hợp? Về cơ bản chúng ta muốn tìm một geohash length nhỏ nhất nhưng vẫn có thể bao trùm được bán kính do user chỉ ra. Dưới đây là mối quan hệ tương ứng giữa bán kính (radius) và geohash length

![IMG_0899](https://user-images.githubusercontent.com/15076665/205430082-c0e1ff00-3324-4458-ad7c-7eeb259d1c09.jpg)

Cách làm này về cơ bản là khá ổn, tuy nhiên có một số trường hợp biên mà ta cần xem xét như dưới đây.

##### Boundary issues

Geohash giả định rằng prefix của các chuỗi geohash càng giống nhau thì vị trí của 2 locations sẽ càng gần nhau như hình dưới đây:

![Screen Shot 2022-12-03 at 16 31 25](https://user-images.githubusercontent.com/15076665/205430173-7e1a8805-f13b-4f47-a354-7bb11f88c4e9.png)

**Vấn đề thứ nhất:**

Giả định trên chỉ đúng 1 chiều chứ không hề đúng cho chiều ngược lại bởi lẽ, 2 địa điểm ở cạnh nhau có thể nằm ở "2 mảng to" khác nhau. Do đó câu truy vấn SQL như dưới đây không thể lấy ra toàn bộ các businesses gần người dùng nhất được.

```SQL
SELECT * FROM geohash_index WHERE geohash LIKE "9q8zn%";
```

**Vấn đề thứ hai:**

Hai location có thể cho chung geohash prefix khá dài tuy nhiên chúng lại nằm ở "2 mảnh khác nhau".

![Screen Shot 2022-12-03 at 16 38 21](https://user-images.githubusercontent.com/15076665/205430411-2e3463d0-792c-41cf-bdc9-efd41e37b9d9.png)

Giải pháp ở đây đó chính là ta không chỉ fetch về các businesses thuộc về grid hiện thời mà còn fetch về từ những grid neighbors xung quanh.

##### Không đủ businesses

Một vấn đề khác có thể phát sinh ở đây đó là việc không có đủ businesses để fetch về dẫn tới không đáp ứng được đúng nhu cầu từ phía user.

Có 2 giải pháp cho vấn đề nêu trên.

- Giải pháp 1: chỉ trả về các businesses trong bán kính tìm kiếm của user, cách này dễ triển khai tuy nhiên vẫn có khả năng không đáp ứng được yêu cầu từ phía user.
- Giải pháp 2: mở rộng bán kính tìm kiếm. Cách triển khai ở đây đó là bỏ bớt đi digit cuối cùng trong geohash để mở rộng phạm vi tìm kiếm, cứ thế cho đến khi số lượng business được trả về lớn hơn so với số lượng kì vọng thì thôi. Minh hoạ như hình bên dưới:

![IMG_0900](https://user-images.githubusercontent.com/15076665/205430594-66a23024-b799-49ba-8872-90d9802ab323.jpg)

#### Giải thuật 4: QuadTree

Một giải pháp khác là sử dụng quadtree. Cách này sử dụng data structure là quadtree chuyên dùng để "làm phẳng" các dữ liệu 2 chiều. Tư tưởng như sau: ta chia không gian thành 4 phần (4 grids) một cách đệ quy, neo đệ quy ở đây đó là trong một grid thì số lượng businesses phải nhỏ hơn 100 (con số này có thể thay đổi tuỳ theo business requirement).

Quadtree **KHÔNG PHẢI LÀ DATABASE SOLUTION** do nó hoạt động ở môi trường in-memory (chạy trong mỗi LBS server) & sẽ được build tại start-up time

Ta giả sử cả thế giới có 200 triệu businesses.

![Screen Shot 2022-12-03 at 16 55 53](https://user-images.githubusercontent.com/15076665/205430946-97de95b5-381d-4cbe-b955-fdcd76ef3d7a.png)

Hình dưới đây sẽ mô tả cụ thể hơn quá trình build tree.

![Screen Shot 2022-12-03 at 17 02 33](https://user-images.githubusercontent.com/15076665/205431159-8847be94-210d-4b62-9b43-3494adcc9ef5.png)

```TS
// pseudo code
const buildQuadTree(TreeNode node) {
  if (countNumberOfBusinessesInCurrentGrid(node) > 100) {
    node.subdivide();
    for (TreeNode child in node.getChilds()) {
      buildQuadTree(child);
    }
  }  
}
```

**Về lượng bộ nhớ sử dụng:**

Dữ liệu lưu ở node lá:

![IMG_0901](https://user-images.githubusercontent.com/15076665/205431909-1a7b0823-ab3c-4616-832a-2c9554f586c8.jpg)

Dữ liệu lưu trong internal node:

![IMG_0902](https://user-images.githubusercontent.com/15076665/205431917-ad1e81c2-b303-4e76-8104-8d4716e6e9bb.jpg)

- 1 grid có thể lưu tối đa 100 businesses
- Số lượng node lá: **200 triệu / 100 = 2 triệu**
- Số lượng internal nodes: **2 triệu x 1/3 = 0.67 triệu**
- Tổng lượng bộ nhớ cần sử dụng: **2 triệu x 832 bytes + 0.67 triệu x 64 bytes ~ 1.71GB**

Memory requirement ở đây là khá nhỏ.

> Điều đáng để nhấn mạnh ở đây không phải là những con số cụ thể về bộ nhớ mà là việc quadtree không tiêu tốn quá nhiều bộ nhớ và có thể được tải bở 1 server duy nhất

Theo như trên không có nghĩa là ta chỉ cần 1 server duy nhất, ta vẫn cần nhiều servers để phục vụ lượng requests khổng lồ từ phía users.

Thời gian cần để build tree **(n/100) x log (n/100)** (với n là số lượng businesses). Do đó chỉ tốn vài phút để build quadtree mà thôi.

**Vậy làm thế nào để có thể tìm các businesses gần nhất bằng quadtree?**

1. Build tree
2. Duyệt cây bắt đầu từ root node cho đến khi xuống đến node lá thể hiện `search origin`. Nếu node có 100 businesses, ta trả về kết quả luôn. Nếu không ta sẽ trả về thêm từ các nodes bên cạnh.

**Các operations khác nên được cân nhắc với quadtree:**

Do quá trình build quadtree được tiến hành tại start-up time, do đó quá trình start-up server sẽ lâu hơn so với bình thường, dẫn đến việc re-deploy sẽ làm server bị offline trong một khoảng thời gian nhất định. Blue/ Green deployment có thể giải quyết vấn đề này, tuy nhiên với lượng dữ liệu đến từ 200 triệu businesses thì việc 1 server phải chịu toàn bộ tải cũng sẽ gây ra rủi ro tương đối cao.

Một yếu tố khác cũng nên được nhắc tới ở đây đó là việc thêm mới, cập nhật, xoá business sẽ làm ảnh hưởng đến cây. Ta có thể tiến hành re-build lại từng tập con server cho đến hết, tuy nhiên làm điều này sẽ dẫn đến việc dữ liệu trả về cho users sẽ bị cũ. Vấn đề có thể được giải quyết bằng cách thoả thuận với phía businesses rằng thông tin sẽ được phản ánh lên hệ thống vào ngày hôm sau.

#### Giải thuật 5: Google S2

Map không gian xuống đường cong Hilbert để từ đó search trên không gian 1 chiều (dựa theo đường cong Hilbert), đây cũng là in-memory solution.

S2 rất ổn khi triển khai geofencing (Geofence là một đường bao địa lý ảo bao quanh 1 khu vực trên thực tế)

![GeoFence](https://user-images.githubusercontent.com/15076665/205433848-8e2e5440-be81-47e1-a2f7-393b274a7def.jpeg)

Geofence cho phép chúng ta bao quát được những địa điểm yêu thích của user, hoặc gửi notification đến các user nằm ngoài khu vực đó.

Một ưu điểm khác của S2 đó là việc nó cho phép chúng ta có thể chỉ định ra:

- Max level
- Min level
- Max cell

chứ không phải theo một quy định cho trước như geohash. Từ đó kết quả trả về từ phía S2 sẽ cụ thể và chi tiết hơn.

**Lời khuyên:**

Trong quá trình phỏng vấn ta nên lựa chọn **geohash** và **quadtree** do chúng đơn giản và dễ giải thích hơn so với S2

#### Geohash vs quadtree

**Geohash:**

- Dễ triển khai.
- Hỗ trợ việc trả về các businesses trong phạm vi bán kính cho trước.
- Geohash length là cố định do đó khó có thể linh hoạt trong việc điều chính kích cỡ của grid
- Update index dễ dàng, ví dụ khi ta muốn loại bỏ một business khỏi grid, ta chỉ việc loại bỏ record tương ứng trong DB là xong.

![IMG_0904](https://user-images.githubusercontent.com/15076665/205434200-85be600d-2127-4735-8434-ed5b7c6c0dd3.jpg)

#### Quadtree

- Khó triển khai hơn so với geohash
- Hỗ trợ tìm k businesses gần nhất. Việc này sẽ hỗ trợ rất nhiều cho user, ta lấy ví dụ với việc user bị hỏng xe giữa đường, chị ta cần tìm **chỗ sửa xe gần mình nhất** chứ không phải **chỗ sửa xe trong phạm vi bán kính R**. Quadtree có thể **tự điều chỉnh query range** để từ đó trả về **k business** cho ta.
- Việc update business khá phiền phức khi phải duyệt cây từ root đến lá (chi phí này là `O(logn)`), hơn thế nữa việc triển khai là khá phức tạp với việc multi-thread đều truy cập đến cây (cần có cơ chế locking). Ngoài ra việc tái cân bằng lại cây cũng rất cần thiết nếu trường hợp node lá không có khả năng thêm business mới.

## Bước 3: Deep Dive design

### Scale the database

Trong phần này ta sẽ nói về việc scale trên 2 bảng quan trọng đó là:

- Business table
- Geospatial index table

#### Business table

Cách làm phổ biến đó là sharding thành nhiều database dựa theo sharding key là `bussiness_id`. Cách làm này cũng rất dễ để maintain trong tương lai.

#### Geospatial index table

Geohash và quadtree đều sử dụng bảng này, với trường hợp của geohash sẽ đơn giản hơn để minh hoạ ở đây. Thông thường 1 geohash có nhều businesses bên trong, ta có thể triển khai cấu trúc bảng theo 2 cách tiếp cận sau:

**Cách 1:**

![Screen Shot 2022-12-03 at 18 53 47](https://user-images.githubusercontent.com/15076665/205434929-501f9960-ba04-44e7-acd8-6a1b210c06c3.png)

Như hình trên ta thấy tương ứng với mỗi `geohash` sẽ là một danh sách các business ids (lưu ở dạng JSON array), tức là mọi business ids sẽ được lưu tại một record duy nhất.

**Cách 2:**

![Screen Shot 2022-12-03 at 18 55 13](https://user-images.githubusercontent.com/15076665/205434997-e83addae-fbf3-44d4-82c3-daa77b768f4e.png)

Cách làm này sẽ chỉ lưu một business_id tạo một record mà thôi, từ đó dẫn đến với nhiều business_ids trong cùng một geohash ta sẽ có table data như sau:

![Screen Shot 2022-12-03 at 18 57 11](https://user-images.githubusercontent.com/15076665/205435099-7e6c5317-dcdf-4f2c-ae76-0e8c3afb9e25.png)

**Lời khuyên:**

Hãy lựa chọn cách làm 2 thay vì cách thứ nhất. Nguyên nhân là bởi cách làm thứ nhất sẽ dẫn đến những điều phiền toái như sau:

- Khi update ta phải duyệt toàn bộ JSON array để tìm ra chỗ cần update
- Khi insert ta cũng phải duyệt toàn bộ JSON array để kiểm tra xem có bị duplicate hay không
- Hơn thế nữa ta cũng cần phải có cơ chế locking để tránh việc cập nhật đồng thời vào JSON array này

Với cách làm thứ 2 ta chỉ cần thêm, cập nhật 1 bản ghi mới hoặc đang có (dễ dàng hơn rất nhiều) với việc kèm theo đó là `compound_key (geo_hash, business_id)`

#### Scale the geospatial index

Bản thân quadtree cũng chỉ tốn 1.71GB nên dữ liệu geohash cũng sẽ chỉ ở mức tương đương, với mức này thì server hoàn toàn có thể tải được. Tuy nhiên trong trường hợp có quá nhiều requests thì việc chia tải cũng là điều cần thiết nếu không đủ CPU và bộ nhớ để xử lí.

Tuy nhiên chia tải có 2 cách:

- Sharding
- Replicating

Việc lựa chọn sharding ở đây là không cần thiết do việc sharding là khá phức tạp, hơn thế nữa sharding cần phải thêm vào tầng application dẫn đến việc triển khai sharding là phức tạp hơn so với replicating.

Nên lời khuyên ở đây là sử dụng `replicating`.

### Caching

Trước khi triển khai cache layer, ta cần cân nhắc xem liệu có nhất thiết phải sử dụng đến cache ở đây không. Câu trả lời là có:

- Workload chủ yếu là read, dataset cũng không phải quá lớn, queries cũng không ảnh hưởng đến I/O quá nhiều do đó chúng nên được chạy ở in-memory cache
- Nếu read performance bị bottleneck, ta có thể thêm các read replicas để cải thiện read performance

Việc sử dụng cache nên được cân nhắc dựa trên:

- Cost analysis
- Business requirement

Sau khi quyết định sử dụng cache, hãy quyết định đển **chiến lược cache**.

#### Cache key

Một trong những cách lựa chọn cache key đó là dựa theo toạ độ (vĩ độ, kinh độ) của người dùng. Tuy nhiên key này có những vấn đề sau:

- Toạ độ gửi từ mobile device của user có thể không chính xác (ngay cả khi user không di chuyển, toạ độ cũng sẽ thay đổi sau mỗi lần fetch về trên mobile device).
- Khi người dùng di chuyển thì toạ độ của người dùng cũng sẽ thay đổi theo, tuy nhiên sự thay đổi này đa phần có thể được bỏ qua.

Do đó toạ độ không thích hợp dùng làm cache key. Trong trường hợp lí tưởng, những sự thay đổi nhỏ về mặt vị trí của người dùng cũng sẽ map đến cùng cache key.

Hai giải thuật geohash/ quadtree ở phần trước đều có thể xử lí vấn đề này tốt bởi vì những businesses trong cùng một grid đều có chung geohash.

#### Các loại dữ liệu sẽ được cache

Có 2 loại dữ liệu sẽ được cache để cải thiện hiệu năng của hệ thống:

![Screen Shot 2022-12-03 at 22 29 12](https://user-images.githubusercontent.com/15076665/205443199-5d4de067-f18e-4e53-8523-0057e56aba84.png)

**List of business ids in the grid:**

Cách làm có thể mô tả như trong pseudo code dưới đây

```TS
const getNearbyBusinessIds = (geohash: string) => {
  const cacheKey = hash(geohash);
  let listOfBusinessIds = Redis.get(cacheKey);

  if (!listOfBusinessIds) {
    listOfBusinessIds = query.executre(`SELECT business_id FROM geohash_index WHERE geohash LIKE ${geohash}`);
    Redis.set(cacheKey, listOfBusinessIds);
  }

  return listOfBusinessIds;
}
```

Khi tiến hành thêm mới, cập nhật, xoá business thì các thao tác này cũng không tốn quá nhiều effort, đồng thời geohash cũng không cần phải sử dụng cơ chế locking nên các thao tác cập nhật cache để phản ánh đúng dữ liệu mới nhất có thể thực hiện một cách dễ dàng.

Theo như yêu cầu thiết kế hệ thống thì người dùng có thể chọn các bán kính `500m`, `1km`, `2km`, `5km` tương ứng với các `geohash_legnth` lần lượt là `4`, `5`, `5` và `6`.

Ta sẽ lần lượt cache search result trong Redis với các key `geohash_4`, `geohash_5`, `geohash_6`

Như đã nói, ta có cả thảy 200 triệu businesses (mỗi business sẽ thuộc về 1 grid), vậy nên ta cần lượng bộ nhớ cụ thể như sau:

- Redis values: **8 byte x 200 triệu x 3 precisions ~ 5GB**
- Redis keys: không đáng kể
- Tổng lượng bộ nhớ cần dùng ở đây là: **~ 5GB**

Với lượng bộ nhớ như trên, ta chỉ cần 1 modern Redis server là đủ, tuy nhiên để đáp ứng lượng request lớn cũng như giảm đi độ trễ khi truy vấn, ta sẽ triển khai Redis cluster global. Các bản sao của cache data sẽ được deploy globally như hình dưới

![Screen Shot 2022-12-04 at 13 02 30](https://user-images.githubusercontent.com/15076665/205473739-1d398a23-251d-4581-bd19-2dfb39fa1f6f.png)

Các thông tin chi tiết về business sẽ lưu theo dạng

```JSON
{
  // key: business_id, value: business detail infor
  "business_id": {
    "bussiness_name": "ABC",
    "business_ranking": "4",
    // ...
  }
}
```

#### Region và AZ

Về bản chất chính là việc ta đưa LBS đến "gần user" hơn.

- Triển khai trên nhiều region, ví dụ: user ở Mỹ sẽ kết nối tới LBS tại Mỹ, ở châu Âu là châu Âu, ...
- Với các khu vực đông đúc ta có thể triển khai 1 region riêng hoặc multiple AZs để phân đều tải, tránh SPOF.
- Để đáp ứng các privacy laws, ta có thể tạo một region riêng cho nơi đó với cơ chế đặc thù.

#### Quá trình tìm các businesses gần nhất

1. User tìm kiếm các nhà hàng trong bán kính 500m, client sẽ gửi thông tin về toạ độ (vĩ độ, kinh độ) cũng như bán kính tìm kiếm tới load balancer.
2. Load balancer sẽ forward đến LBS.
3. LBS sẽ dựa theo bán kính tìm kiếm của user để đưa ra `geohash_length` phù hợp nhất (ở đây là 6).
4. LBS sẽ tính các neighboring geo hash và thêm vào danh sách các geohashes, kết quả sẽ như sau: `[my_geohash, neighbor1_geohash, neighbor2_geohash, ..., neighbor3_geohash]`.
5. Tương ứng với từng geohash trong danh sách các geohashes trên, LBS sẽ lấy dữ liệu ra từ Redis cluster (bao gồm các business IDs), qúa trình lấy dữ liệu này có thể được triển khai song song để giảm đi độ trễ.
6. Dựa theo business IDs lấy ra được, LBS sẽ fetch dữ liệu về business từ `Redis business data`, tính toán khoảng cách tới user, ranking, ... và trả về kết quả cho user.

#### View, cập nhật, thêm mới hoặc xoá business

Việc xem thông tin chi tiết về business sẽ dựa trên cơ chế cache. Cụ thể là nếu trong cache có thông tin về business, thông tin sẽ được trả về cho client luôn. Nếu không thì LBS sẽ fetch dữ liệu từ Database cluster, lưu nó xuống Redis cache rồi trả về cho client.

Với việc cập nhật, thêm mới hoặc xoá business thì như đã thoả thuận với business, các thông tin mới sẽ được phản ánh vào ngày hôm sau. Do đó ta cần một `nightly job` để có thể đồng bộ hoá dữ liệu mới từ database sang Redis cache.
