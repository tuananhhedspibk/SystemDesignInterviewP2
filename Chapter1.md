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
