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
