# Hotel Reservation System

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Hệ thống đặt lịch phòng khách sạn thường khá phức tạp và đa dạng tuỳ theo business model. Do đó trước khi thiết kế, ta cần làm rõ yêu cầu với interviewer.

- Hệ thống phải phục vụ cho một chuỗi khách sạn với 5000 khách sạn và tổng số là 1 triệu phòng.
- Người dùng sẽ phải trả toàn bộ chi phí khi tiến hành đặt phòng.
- Người dùng có thể đặt phòng qua web hoặc app.
- Người dùng có thể cancel lịch đặt phòng của họ.
- Hệ thống sẽ đáp ứng 10% overbooking (tức là khách sạn sẽ cho phép thuê "nhiều phòng" hơn số phòng họ hiện có, mục đích ở đây là xử lí những case khách hàng cancel lịch hẹn).
- Do thời gian thiết kế có hạn nên sẽ chỉ tập trung vào những tính năng như phía dưới đây:

1. Hotel-related page.
2. Hotel room-related detail page.
3. Đặt phòng.
4. Admin panel để thêm/ sửa/ xoá thông tin phòng.
5. Hỗ trợ overbooking.

- Giá thuê phòng sẽ thay đổi tuỳ theo ngày.

### Non-functional requirements

- Hỗ trợ high-concurrency. Vào các mùa "cao điểm" du lịch hoặc sự kiện lớn, các khách sạn nổi tiếng sẽ có nhiều khách hàng cùng book phòng.
- Độ trễ vừa phải. Sẽ là lí tưởng nếu hệ thống có thể hồi đáp lại phía user nhanh nhất có thể thế nhưng độ trễ vài giây cũng có thể được coi là chấp nhận được.

### Back-of-the-envelope estimation

- 5000 khách sạn và tổng số 1 triệu phòng.
- Giả sử 70% phòng đã được đặt và thời gian sử dụng trung bình là 3 ngày.
- Lượng đặt phòng theo ngày = (1 triệu \* 0.7 / 3) = 233,333 ~ 240,000
- Lượng đặt phòng theo giây: 240,000 / 10^5 = 3 (TPS không quá cao).

Giờ chúng ta sẽ tính toán QPS cho mọi pages. Có 3 bước trong follow điển hình của người dùng như sau:

1. Xem thông tin khách sạn/ phòng (query).
2. Xem trang đặt phòng. Người dùng có thể xác nhận các thông tin chi tiết về việc đặt phòng như (ngày, số lượng khách, payment infor) (query).
3. Đặt phòng, user click vào nút đặt phòng (transaction).

Ta giả sử khoảng 10% users sẽ đi đến bước tiếp theo và 90% users sẽ bỏ giữa chừng flow nói trên.

![Screenshot 2024-03-09 at 13 38 52](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/661f9167-0a28-4244-a10e-ea91e31fa2b5)

Hình trên đây sẽ cho thấy bản estimate "thô" về QPS, ở đây ta lần ngược từ giá trị **Lượng đặt phòng theo giây** ở trên.

## Bước 2: High-level design

### API design

Với hệ thống đặt phòng này, ta sẽ thiết kế theo hướng RESTful API.

![Screenshot 2024-03-09 at 16 28 13](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/1e53d81b-8114-4f23-9f6e-34fe70ba1c55)

![Screenshot 2024-03-09 at 16 29 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/4fdedac9-0acf-4ce8-aa14-d7f6a38f74b9)

![Screenshot 2024-03-09 at 16 29 42](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/7900d4a0-886c-42ed-b20d-87a4a020a80e)

Chú ý rằng `reservation ID` sẽ được sử dụng như idempotency key để tránh double book (ở đây mang ý nghĩa là nhiều lần đặt cùng 1 phòng tại cùng 1 thời điểm).

### Data model

Với hệ thống đặt phòng khách sạn, chúng ta cần support những câu queries sau:

- Query 1: Xem thông tin chi tiết về khách sạn.
- Query 2: Tìm các loại phòng hiện có trong một khoảng thời gian nhất định.
- Query 3: Đặt phòng.
- Query 4: Tìm lịch sử đặt phòng.

Theo như yêu cầu ở phần **back-of-envelope estimate**, ta thấy rằng quy mô hệ thống không quá lớn, chỉ cần đáp ứng được những đợt cao điểm du lịch là đủ, do đó ta sẽ sử dụng RDB ở đây với những lí do như sau:

- **RDB hoạt động tốt với read-heavy và write less** do trong thực tế số lượng những người thực sự đặt phòng ít hơn rất nhiều so với những người xem thông tin về khách sạn và phòng (NoSQL hoạt động tốt hơn với các ứng dụng ghi nhiều hơn đọc).
- RDB cung cấp ACID (atomicity, consitency, isolation, durability), ACID sẽ phát huy sức mạnh với các hệ thống reservation kiểu này do nó xử lí tốt các vấn đề như **negative balance**, **double charge**, **double reservations**, ... ACID sẽ giúp cho application code đơn giản hơn rất nhiều.
- RDB cũng dễ dàng giúp mô hình hoá dữ liệu. Do cấu trúc của business data đã rất rõ ràng và mối quan hệ giữa các entities (hotel, room, room_type, ...) cũng ổn định.

![Screenshot 2024-03-09 at 23 18 56](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/6499b7b7-c27a-4500-9cdb-9339113b3d1c)

Do bản thân các thuộc tính của các bảng nói trên đã tự định nghĩa nó là gì, nên ở đây chúng ta chỉ đi sâu vào cột `status` của bảng `room` với các giá trị như sau:

![Screenshot 2024-03-09 at 23 21 09](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c0de77c5-d471-478d-ad60-ebc48713b25d)

Thực ra data model phía trên sẽ thích hợp cho Airbnb do khi người dùng tiến hành đặt phòng họ đã phải biết chính xác mình chọn phòng nào nên việc sử dụng `room_id` là hoàn toàn hợp lý. Nhưng với hệ thống đặt phòng của khách sạn thì người dùng chỉ quan tâm dến **loại phòng** thay vì chính xác phòng nào (phòng nào sẽ được quyết định khi khách hàng làm thủ tục check-in chứ không phải thời gian đặt phòng).

### High-level design

Chúng ta sẽ sử dụng kiến trúc micro-service ở đây.

![Screenshot 2024-03-10 at 11 47 29](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/854aa6e1-6a91-451f-bcdc-3c4cce3ffeba)

Chúng ta sẽ đi phân tích một vài components chính có trong thiết kế trên.

- `Public API Gateway`: hỗ trợ rate-limiting, authentication, ... Ngoài ra api gateway cũng redirect request đến service tương ứng tuỳ theo endpoint mà nó nhận được, ví dụ: request load các thông tin về khách sạn sẽ được điều hướng đến `hotel service`, request đặt phòng sẽ được điều phối đến `reservation service`.
- `Interal APIs`: các API này chỉ có thể được sử dụng bởi các authorized staff, chúng được truy cập thông qua các internal sofware hoặc websites. Thường sẽ được bảo vệ bởi VPN.
- `Hotel service`: cung cấp các thông tin liên quan đến khách sạn hoặc phòng, đây đa phần đều là các static data nên có thể cache lại được.
- `Rate service`: cung cấp thông tin về giá phòng, có một sự thật trong ngành công nghiệp khách sạn đó là khi số lượng phòng càng ít thì giá phòng sẽ lại càng cao.

Trong thực tế giữa các services sẽ có sự tương tác qua lại, ví dụ như hình dưới đây:

![Screenshot 2024-03-10 at 11 58 06](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/977320f8-b22c-4963-9c6f-b197603aee17)

Giữa `Rate service` và `Reservation service` sẽ có sự tương tác qua lại do `Reservation service` cần biết giá phòng để tính toán giá tiền mà người dùng phải trả. Hoặc khi `Hotel manangement service` thay đổi thông tin liên quan đến giá, phòng, ... thì những sự thay đổi này cần phải được cập nhận đến các services như `Hotel service` và `Rate service`.

Về việc tương tác giữa các inter-service, ta sẽ sử dụng RPC (remote procedure call) cũng như các framework như gRPC.

### Deep-Dive Design

Trong phần này chúng ta sẽ đi sâu vào các vấn đề sau:

- Cải thiện data model.
- Concurrency issues.
- Scaling system.
- Giải quyết vấn đề dữ liệu không thống nhất trong kiến trúc micro-service.

#### Cải thiện data model

Như đã nói trong phần high-level design, khi người dùng đặt phòng, họ chỉ lựa chọn và "đặt loại phòng" chứ không phải là một phòng cụ thể nào cả. Vậy ta cần thay đổi API và schema như thế nào ?

Với reservation API, `roomID` sẽ được thay bởi `roomTypeID` trong request parameter. API đặt phòng sẽ trông như thế này.

```json
// POST /v1/reservations

{
  "startDate": "2021-04-28",
  "endDate": "2021-04-30",
  "hotelID": "245",
  "roomTypeID": "123456789",
  "reservationID": "13422445"
}
```

Schema mới sẽ như sau:

![Screenshot 2024-03-11 at 7 55 17](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a1251ff9-73e0-496e-b584-383a48d80ece)

Chúng ta sẽ phân tích một vài bảng quan trọng như sau:sau

- `room`: thông tin về phòng.
- `room_type_rate`: thông tin về giá của từng loại phòng.
- `reservation`: lưu thông tin đặt phòng.
- `room_type_inventory`: lưu thông tin về việc phân bổ phòng. Bảng này có các cột như sau:
  - hotel_id: ID của khách sạn.
  - room_type_id: ID của loại phòng.
  - date: thông tin về ngày.
  - total_inventory: tổng số phòng trừ đi số phòng đang được chưng dụng tạm thời (có thể cho mục đích dọn dẹp, ...).
  - total_reserved: tổng số phòng đã được đặt.

Việc quản lí các bản ghi trong bảng `room_type_inventory` theo ngày sẽ giúp việc query dữ liệu theo date range trở nên dễ dàng hơn.

Như trong hình mô tả schema trên, **(hotel_id, room_type_id, date)** là **composite primary key**. Chúng ta sẽ có daily jobs để tạo ra các inventory data cho tương lai (trong 2 năm).

Giờ chúng ta sẽ thử estimate lượng dữ liệu sẽ được lưu trữ trong tương lai, như ở phần **back-of-the-envelope estimate** ta sẽ có 5000 khách sạn, mỗi khách sạn sẽ có 20 loại phòng nên số lượng bản ghi sẽ là:

> 5000 x 20 x 2 (năm) x 365 (ngày) = 73 triệu

73 triệu không phải là một con số quá lớn đối với single database, tuy nhiên single server đồng nghĩa với SPOF. Để tăng tính sẵn có cho hệ thống, ta sẽ tạo ra thêm replication trên nhiều regions hoặc AZ.

Bảng **room_type_inventory** sẽ kiểm tra xem một người dùng có khả năng đặt phòng hay không. Input và Output sẽ như sau:

- Input: `startDate (2021-07-01)`, `endDate (2021-07-03)`, `roomTypeId`, `hotelId`, `numberOfRoomsToReserve`.
- Output: `True` nếu inventory data đủ chỗ chứa, `False` ngược lại.

Nếu nhìn từ quan điểm SQL, ta sẽ có 2 bước sau:

1. SELECT dữ liệu trong date range

```sql
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
```

![Screenshot 2024-03-12 at 7 58 06](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/37954a61-3975-4ad6-bcea-3c1df4891bce)

2. Với mỗi record, application sẽ kiểm tra điều kiện sau

```ts
if (total_reserved + numberOfRoomsToReserver <= total_inventory) {
}
```

Nếu với mọi records, kết quả là `true` thì có nghĩa là còn phòng cho mọi ngày trong date range đó.

Với điều kiện support 10% overbooking, ta có thể triển khai dễ dàng với schema mới như sau:

```ts
if (total_reserved + numberOfRoomsToReserver <= total_inventory * 1.1) {
  // 1.1 ~ 110%
}
```

Trong trường hợp reservation data quá lớn, dưới đây sẽ là một vài giải pháp:

- Chỉ lưu reservation data hiện thời và trong tương lai, các data cũ sẽ bị archived hoặc đưa vào "cold storage".
- Database sharding, các câu queries thường dùng bao gồm "tạo reservation" và "tìm reservation bằng tên", cả hai câu queries loại này đều cần đến `hotel_id` do đó ta có thể lựa chọn `hotel_id` như là một sharding key, dữ liệu sẽ được shared bằng công thức `hash(hotel_id) % number_of_servers`.

#### Concurrency issues

Một vấn đề khác ở đây cần phải giải quyết đó là "double booking", cụ thể như sau:

1. Một user click nút đặt phòng nhiều lần.
2. Nhiều users cùng đặt một phòng tại cùng thời điểm.

Với kịch bản đầu tiên, ta có thể mô phỏng lại như hình dưới đây

![Screenshot 2024-03-12 at 8 22 51](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/6ec3b00a-f696-42ef-8177-7da711e695e5)

Có hai cách tiếp cận để giải quyết vấn đề này:

- Phía client sẽ disable button sau khi nó được click, tuy nhiên cách làm này không có độ tin cậy cao do client có thể disable đi JS.
- Idempotent APIs, thêm idempontency key trong reservation API request. Một API call được coi là idempotent chỉ khi cho dù nó được gọi nhiều lần thì kết quả vẫn giống nhau.

Hình dưới đây sẽ mô tả việc sử dụng `reservation_id` để tránh double-reservation issue.

![Screenshot 2024-03-12 at 9 15 29](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/570ed64c-a3b5-4cc0-a0f0-c8cd21e84788)

1. Generate reservation order, sau khi người dùng nhập thông tin chi tiết về reservation reservation (room type, check-in date, check-out date) và click submit button, reservation order sẽ được gen bởi reservation service.service
2. Hệ thống sẽ gen ra reservation order cho người dùng review, đồng thời `reservation_id` cũng sẽ được gen bởi `global unique ID generator`.
3. Bước 3
   3.a. Submit reservation 1, `reservation_id` sẽ được coi như một phần của request, chúng ta cũng không nhất thiết phải sử dụng `reservation_id` như một idempotency key, thế nhưng trong thiết kế lần này nó hoạt động tốt và ta cũng sẽ sử dụng `reservation_id` như một primary key cho bảng reservation.
   3.b. Nếu user click nút submit lần 2, reservation 2 được submit đi, thế nhưng do `reservation_id` đã được sử dụng làm primary key của bảng reservation nên điều kiện này sẽ đảm bảo việc không có **double reservation**.

![Screenshot 2024-03-13 at 8 25 46](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ed12400b-572a-43ab-be7c-39deb8f3d8e3)

Với kịch bản thứ hai: điều gì sẽ xảy ra khi có nhiều users cùng đặt một loại phòng vào cùng một thời điểm trong khi chỉ còn duy nhất 1 phòng.

![Screenshot 2024-03-14 at 7 53 37](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/9e6cd0cf-0c40-4a55-b7ff-fa00e7378146)

1. Giả sử database isolation level không phải là serializable. User 1 và User cùng đặt một loại phòng ở cùng một thời điểm nhưng chỉ còn duy nhất 1 phòng còn lại. Ta sẽ có `transaction 1`, `transaction 2` lần lượt tương ứng với xử lí của hai users, lúc này có 100 phòng trong khách sạn và có 99 phòng đã được đặt.
2. `Transaction 2` sẽ kiểm tra xem có đủ phòng còn lại không bằng phép `if(total_reserved + rooms_to_book) <= total_inventory`. Do còn lại 1 phòng nên trả về `True`.
3. `Transaction 1` cũng sẽ kiểm tra xem có đủ phòng còn lại không bằng phép `if(total_reserved + rooms_to_book) <= total_inventory`. Do còn lại 1 phòng nên trả về `True`.
4. `Transaction 1` đặt phòng và cập nhật inventory: `reserved_room` sẽ có giá trị là `100`.
5. `Transaction 2` đặt phòng. Do đặc tính **isolation** của ACID (tức là mọi thay đổi của transaction sẽ không được nhìn thấy bởi các transaction khác cho dến khi transaction được committed), do đó `transaction 2` sẽ vẫn thấy `total_reserved = 99` và `transaction 2` vẫn có thể đặt được phòng, sau đó nó sẽ cập nhật giá trị của `reserved_room` thành `100`. Kết quả này cho phép hai users cùng đặt được phòng dù chỉ còn 1 phòng duy nhất.
6. `Transaction 1` commit thành công.
7. `Transaction 2` commit thành công.

Để giải quyết vấn đề này ta cần có cơ chế "lock", một vài kĩ thuật "lock" tiêu biểu sẽ như sau:

- Pessimistic locking
- Optimistic locking
- Database constrants

Trước khi đi vào giải quyết vấn đề, chúng ta cùng xem xét mã giả SQL khi tiến hành đặt phòng

```SQL
-- Step 1: check room inventory
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} AND ${endDate};

-- For every entry returned from step 1
if ((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback;
}

-- Step 2: Reserve rooms
UPDATE room_type_inventory
SET tota_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate};

Commit;
```

**Pessimistic locking:**
Còn được gọi là pesimistic concurrency control, ngăn việc cập nhật đồng thời bằng cách đặt lock vào record ngay khi user bắt đầu tiến hành cập nhật record. User khác muốn cập nhật thì phải chờ đến khi lock được released (mọi sự thay đổi đã được commit).

Vơí MySQL, `SELECT ... FOR UPDATE` hoạt động bằng cách lock các rows trả về từ câu select, giả sử transaction 1 sẽ bắt đầu trước, lúc này các transactions khác sẽ phải chờ cho đến khi transaction 1 hoàn tất nhiệm vụ của mình thì mới được thực thi.

![Screenshot 2024-03-15 at 8 01 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a781b1ce-37d0-4777-9042-f786a820986c)

Ở hình trên, câu `SELECT ... FOR UPDATE` của transaction 2 sẽ phải chờ vì các rows đã bị lock bởi transaction 1, sau khi transaction 1 kết thúc thì transaction 2 mới hoạt động, lúc này `total_reserved = 100` nên transaction 2 phải rollback.

**Ưu điểm**:

- Tránh việc ứng dụng cập nhật các dữ liệu đang được thay đổi.
- Dễ dàng triển khai và nó tránh được conflict do đã "tuần tự hoá" thứ tự cập nhật, Pessimistic locking rất hữu dụng khi sự "tranh chấp" dữ liệu xảy ra thường xuyên.

**Nhược điểm**:

- Deadlock có thể xảy ra khi nhiều resources được locked, viết một deadlock-free application không phải là một điều dễ dàng.
- Cách làm này khá khó để scale do có thể phát sinh trường hợp transaction lock dữ liệu trong một khoảng thời gian dài dẫn đến các transactions khác không thể truy cập được resources từ đó làm ảnh hưởng đến hiệu năng của DB đặc biệt là khi transaction `long-lived` hat tác động đến nhiều entities.

Do đó chúng ta **KHÔNG NÊN** sử dụng pessimistic locking cho reservation system.

**Optimisitc locking**: còn được gọi là **optimistic concurrency control**, cho phép nhiều users có thể đồng thời update cùng một resource.

Có hai cách triển khai đó là:

- Timestamp
- Version number (đây là cách làm được ưa chuộng hơn do đồng hộ hệ thống có thể không chính xác ở một vài thời điểm)

![Screenshot 2024-03-15 at 10 47 38](https://github.com/tuananhhedspibk/NewAnigram-Infrastructure/assets/15076665/1cbd19e6-b78c-4e62-982a-abe86e33dc1d)
![Screenshot 2024-03-15 at 10 56 09](https://github.com/tuananhhedspibk/NewAnigram-Infrastructure/assets/15076665/6310170b-2dc8-44b3-88a6-79fddf249c68)

1. Column với tên `version` được thêm vào bảng.
2. Trước khi user chỉnh sửa row, user sẽ phải đọc version number.
3. Khi cập nhật dữ liệu, version number sẽ được tăng lên 1 và cập nhật ngược lại vào DB.
4. DB validation check nên được thực thi ở đây khi version tiếp theo nên hơn version hiện tại 1 đơn vị. Transaction abort nếu validation fail và user sẽ phải thử lại bước 2

Optimistic locking thường nhanh hơn so với Pessimistic locking do nó không lock DB thế nhưng khi con-currency cao thì hiệu năng của optimistic locking sẽ bị giảm đi đáng kể.

Ta cùng xem xét trường hợp có nhiều client cùng truy cập hệ thống, khi này sẽ có nhiều user cùng đặt một loại phòng ở cùng một thời điểm (do số lượng client có thể truy cập vào hệ thống là không có giới hạn).

Các clients sẽ nhận về cùng một `version number` và cùng một `total_reserved`, thế nhưng trong số đó chỉ có DUY NHẤT MỘT CLIENT LÀ THÀNH CÔNG, các clients còn lại sẽ fail ở pha validation. Các fail users sẽ phải retry, và trong những lần retries này cũng sẽ chỉ có duy nhất một client là thành công, cứ thế cứ thế lặp đi lặp lại, kết quả cuối cùng là THÀNH CÔNG nhưng việc phải retries lại nhiều lần sẽ đem lại một trải nghiệm tồi cho user.

**Ưu điểm**:

- Ngăn chặn được việc ứng dụng cập nhật những dữ liệu "cũ".
- Nếu nhìn từ quan điểm phía DB, ta sẽ KHÔNG LOCK DATABASE thế nhưng việc xử lí conflict version number này lại diễn ra ở phía ứng dụng.
- Optimisitic locking phát huy tốt khi tần suất "cạnh tranh" dữ liệu thấp, thế nhưng khi nó xảy ra một cách "thường xuyên" thì transaction có thể kết thúc mà không cần phải mất chi phí cho việc quản lí lock.

**Nhược điểm**:

- Hiệu năng rất tồi khi sự "cạnh tranh" về mặt dữ liệu cao.

**Database constraints**: cách tiếp cận này khá "tương đồng" với optimistic locking, nó cài thêm một "ràng buộc" như sau:

```sql
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
```

![Screenshot 2024-03-15 at 15 47 38](https://github.com/tuananhhedspibk/NewAnigram-Infrastructure/assets/15076665/21e1330b-fc10-46c3-aa89-d17876fb0223)

**Ưu điểm**:

- Dễ dàng triển khai.
- Hoạt động tốt khi sự "cạnh tranh" dữ liệu nhỏ

**Nhược điểm**:

- Cũng tương tự như optimistic locking, tỉ lệ fail sẽ tăng khi sự "cạnh tranh" về mặt dữ liệu là đáng kể, khi đó user dù nhìn thấy "còn phòng" nhưng khi bấm nút "Đặt phòng" thì lại thu được kết quả "đặt phòng không thành công", từ đó làm ảnh hưởng đến trải nghiệm của người dùng.
- Database constraint không hề dễ dàng quản lí bên phía application.
- Không phải mọi DB đều hỗ trợ constraints, điều này cần đặc biệt lưu ý khi migrate sang một database solution khác.

Do kĩ thuật này không quá khó triển khai cũng như trên thực tế các "reservation system" thường có "data contention" không quá cao (low QPS) nên đây cũng là một sự lựa chọn không hề tồi.

### Scalability

Thông thường, các hotel reservation system thường có load không quá cao, do đó trên thực tế việc scaling hệ thống cũng không nhất thiết phải quá chú trọng, thế nhưng interviewer có thể hỏi các câu hỏi như "Nếu hệ thống áp dụng cho các page lớn như booking.com hay expedia.com thì việc scaling sẽ diễn ra như thế nào ?"

Trong thực tế, khi hệ thống được scale, ta cần nắm rõ **PHẦN NÀO CỦA HỆ THỐNG SẼ TRỞ THÀNH BOTTLENECK** để từ đó có thể đưa ra giải pháp phù hợp. Do servers của chúng ta là stateless nên việc scaling server khá đơn giản khi ta chỉ cần thêm servers mới là xong. Nhưng database là stateful nên việc scaling database không chỉ đơn thuần là thêm instances mới.

#### Database sharding

