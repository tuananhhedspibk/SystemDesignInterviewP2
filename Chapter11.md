# Payment System

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Payment system có thể được hiểu theo nhiều hướng khác nhau:

- Wallet: `Apple Pay` hoặc `Google Pay`.
- Backend System xử lí các giao dịch thanh toán như `Paypal` hoặc `Stripe`.

Một trong những điều quan trọng nhất cần làm đầu tiên đó là định nghĩa rõ về yêu cầu của hệ thống.

Yêu cầu hệ thống lần này là:

- Giả sử ta sẽ thiết kế payment system tích hợp với một trang EC là Amazon.com, khi khách hàng đặt mua đồ trên Amazon.com, payment system sẽ xử lí toàn bộ các giao dịch chuyển tiền của trang EC.
- Hệ thống sẽ hỗ trợ `Credit card`, `Paypal`, `Bank Card` - nhưng để đơn giản ta chỉ xét với `Credit Card`.
- Thay vì tự mình xử lí các giao dịch liên quan đến credit card ta sẽ sử dụng các `third-party payment processors` như `Stripe` hay `Braintree`, ...
- Chúng ta sẽ không lưu các thông tin liên quan đến credit card trong hệ thống mà sẽ "dựa vào" dịch vụ của bên thứ ba.
- Sẽ có khoảng 1 triệu "giao dịch" mỗi ngày.
- Cần hỗ trợ `pay-out flow` - giống như cách Amazon trả tiền cho các seller hàng tháng.
- Hệ thống sẽ tương tác với rất nhiều internal-services cũng như external-services nên trong trường hợp một service fail, ta cần có giải pháp để giải quyết sự **không thống nhất** về mặt **trạng thái** cũng như **dữ liệu** giữa các services.

Từ yêu cầu phía trên ta có thể liệt kê ra được những yêu cầu mà hệ thống lần này sẽ đáp ứng.

### Yêu cầu về chức năng

- Pay-in flow: payment system sẽ nhận tiền từ customer thay cho seller
- Pay-out flow: payment system sẽ gửi tiền cho các sellers

### Các yêu cầu khác ngoài chức năng

- Khả năng chịu lỗi, xử lí các giao dịch "thất bại" một cách cẩn thận.
- Đảm bảo sự đồng bộ, nhất quán giữa các **internal services (payment service, accounting service)** và **external services (payment service providers - PSP)**

### Back-of-the-envelope estimation

1 ngày cần xử lí 1 triệu transaction (giao dịch), tức là khoảng

```txt
1,000,000 / 10^5 = 10 transaction/s (TPS)
```

10 transaction/s không phải là một con số quá lớn nên do đó ta không cần quan tâm quá nhiều đến thông lượng của DB mà cần tập trung vào **payment transaction**

## Bước 2: High-level design

Ở high-level ta sẽ chia flow của hệ thống thành 2 luồng chính như sau:

- Pay-in flow
- Pay-out flow

Lấy ví dụ với trang EC là Amazon

1. Người dùng mua hàng.
2. Người dùng trả tiền, tiền sẽ đi đến **tài khoản của Amazon** (pay-in flow).
3. Amazon sẽ chỉ giữ lại một khoản phí nhất định.
4. Khi hàng đã đến tay người dùng, Amazon sẽ chuyển số tiền sau tính phí đến cho seller (pay-out flow).

![Screen Shot 2023-10-19 at 21 58 18](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/b028ed52-3da6-4e42-aefd-3af4435570d6)

Có thể mô tả lại flow của hệ thống như hình phía trên.

### Pay-in flow

![Screen Shot 2023-10-19 at 22 37 48](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/e0ea5a99-ad28-4617-a95b-7b48f1042fbc)

Bước 2 đó là khi payment service lưu payment event trong DB.

Bước 4 đó là khi payment executor lưu payment order trong DB.

Bước 6 đó là sau khi payment executor xử lí thành công payment thì payment service sẽ cập nhật wallet để ghi nhận seller có bao nhiêu tiền (seller balance).

#### Payment Service

Thông thường, payment service sẽ chỉ nhận `payment events` từ users và tiến hành quá trình xử lí payment. Thế nhưng payment service chỉ xử lí các `payment events` đã pass **risk check** (risk check ở đây bao gồm việc kiểm tra có phải các giao dịch mờ ám hay không, ...). Quá trình **risk check** này thường được đảm nhận bởi một bên thứ ba do nó khá phức tạp.

#### Payment Executor

Thực thi một `payment order` thông qua PSP (Payment Service Provider như paypal, stripe, ...)
Một `payment event` có thể bao gồm nhiều `payment orders`

#### Payment Service Provider (PSP)

PSP sẽ chuyển tiền giữa các tài khoản với nhau.

#### Card Schemes

Xử lí các credit card operation. Ví dụ như: visa, master-card

#### Ledger

Lưu financial record của payment transaction. Ví dụ khi user trả $1, ta sẽ lưu:

- debit $1 từ user
- credit $1 cho seller

Ledger system có vai trò quan trọng trong:

- Post-payment analysis
- Phân tích doanh thu

#### Wallet

Lưu account balance hoặc tổng số tiền mà user đã trả.

### APIs cho payment service

Ta sử dụng Restful API ở đây.

#### POST /v1/payments

API này đảm nhận việc thực thi một payment event.

Các thông tin nó cần:

- buyer_info
- checkout_id: Global Unique ID cho lần checkout này.
- credit_card_info: thông tin credit card đã mã hoá hoặc payment token (giá trị này là do PSP chỉ định).
- payment_orders: danh sách các payment orders thuộc về payment event lần này.

**payment order** sẽ gồm các thông tin sau:

- seller_account
- amount: lượng tiền.
- currency: đơn vị tiền tệ.
- payment_order_id: global unique ID

Chú ý rằng **payment_order_id** là global unique, khi payment executor gửi payment request đến PSP, **payment_order_id** sẽ được sử dụng như là một **deduplication ID - idempotency key**.

Chú ý nữa đó là các đơn vị số như **amount** vẫn sẽ để là string vì tuỳ theo protocol, software mà giá trị double hoặc giá trị số có thể sẽ thay đổi.

> Khi lưu trữ hoặc tương tác giữa các services ta nên dùng string, chỉ chuyển đổi sang number khi hiển thị hoặc tính toán

#### GET /v1/payments/:id

API này sẽ trả về execution status của một payment order dựa theo `payment_order_id`

### Data model cho payment service

Về cơ bản ta cần 2 bảng:

- Payment Event
- Payment Order

Đối với payment system, performance không phải là yếu tố quan trọng nhất cần cân nhắc mà là:

1. Tính ổn định: lựa chọn những storage được các hệ thống payment lớn tin dùng trong 1 khoảng thời gian dài (VD: 5 năm trở lên).
2. Có nhiều tool hỗ trợ đi kèm: monitoring hoặc investigation.
3. Có thể dễ dàng tuyển dụng được các DBA (database administrator).

Thông thường ta sẽ sử dụng relation database truyền thống với ACID transaction thay vì NoSQL.

**Payment Event** sẽ bao gồm các thông tin sau:

- checkout_id: string **PK**
- buyer_info: string
- seller_info: string
- credit_card_info: tuỳ theo card provider
- is_payment_done: boolean

**Payment Order** sẽ bao gồm các thông tin sau:

- payment_order_id: string **PK**
- buyer_account: string
- amount: string
- currency: string
- checkout_id: string **FK**
- payment_order_status: string
- ledger_updated: boolean
- wallet_updated: boolean

Chúng ta sẽ cùng đi phân tích 2 bảng trên

- **checkout_id** là khoá ngoại, một lần checkout sẽ tạo ra **1 payment event** có thể bao gồm **nhiều payment orders**
- Khi gọi third-party PSP để trừ tiền từ tài khoản của buyer, tiền sẽ không trực tiếp đi vào tài khoản của seller mà sẽ đi vào tài khoản của trang EC trước (quá trình này gọi là pay-in), sau khi điều kiện để chuyển tiền sang tài khoản của seller được thoả mãn mãn(ví dụ như việc hàng đã đến tay buyer) thì tiền sẽ chuyển từ tài khoản của EC sang seller (quá trình này gọi là pay-out). Do đó trong suốt quá trình diễn ra `pay-in` ta không nhất thiết phải quan tâm đến **seller account** mà chỉ cần quan tâm đến **buyer account** là đủ.

Trong bảng payment-order ở trên, cột `payment_order_status` sẽ có kiểu dữ liệu là enum với các giá trị đã được định nghĩa trước gồm (NOT_STARTED, EXECUTING, SUCCESS, FAILED), logic update nó sẽ diễn ra như sau:

1. Giá trị khởi tạo ban đầu của `payment_order_status` luôn là `NOT_STARTED`.
2. Khi `payment-service` gửi `payment order` sang cho `payment-executor`, giá trị của `payment_order_status` sẽ là `EXECUTING`.
3. `payment-service` sẽ cập nhật `payment_order_status` thành `SUCCESS` hoặc `FAILED` tuỳ thuộc vào kết quả trả về từ `payment-executor`.

Khi `payment_order_status` được cập nhật thành `SUCCESS`, payment-service tiếp theo sẽ gọi đến `wallet-service` để cập nhật seller balance account, đồng thời cập nhật giá trị của `wallet_updated` thành `TRUE` (để đơn giản hoá ta giả sử rằng wallet-service luôn thành công).

Sau khi quá trình gọi `wallet-service` kết thúc, `payment-service` sẽ gọi `ledger-service` để cập nhật ledger DB, sau đó sẽ cập nhật `ledger_updated` thành `TRUE`.

Khi mọi payment-order với cùng một `checkout_id` được xử lí thành công, `payment-service` sẽ cập nhật `is_payment_done` của `payment-event` thành `TRUE`.

Sẽ có một cron-job check trạng thái của payment-order định kì và sẽ gửi alert đến cho dev khi một payment-order mãi không kết thúc được.

### Double-entry ledger system

Có một khái niệm rất quan trọng trong thiết kế payment system đó là **double-entry principle** (còn được gọi là double-entry accounting/bookkeeping).

Nó sẽ ghi nhận mọi transaction thành 2 records khác nhau tương ứng với 2 ledger accounts khác nhau với cùng một lượng tiền. Một account sẽ bị rút tiền, một account sẽ nhận được tiền. Ví dụ:

| Account | Debit | Credit |
| ------- | ----- | ------ |
| buyer   | $1    |        |
| seller  |       | $1     |

Double-entry system sẽ đảm bảo rằng tổng của mọi transaction entries sẽ luôn là 0. Nếu bị lệch 1 đồng thì chứng tỏ một ai đó đã "lấy cắp được" 1 đồng đó.

Việc này đảm bảo khả năng truy vết, tính thống nhất trong suốt vòng đời của payment.

### Hosted payment page

Hầu như mọi services đều lựa chọn việc không lưu trữ thông tin về credit card trong hệ thống của mình vì nếu muốn làm điều đó họ cần phải đảm bảo rất nhiều điều kiện ràng buộc với các cơ quan chức năng liên quan.

Thay vào đó các services sẽ sử dụng `hosted credit card page` được cung cấp bởi PSPs.

Với website đó có thể là: widget hoặc iframe.

Với app thì đó có thể là SDK.

Bản thân `hosted payment page` được PSPs cung cấp khả năng lấy được thông tin về thẻ của khách hàng một cách trực tiếp thay vì dựa vào payment-service của chúng ta.

### Pay-out flow

Flow của phần này cũng không khác nhiều so với pay-in flow ngoại trừ việc pay-out flow sẽ sử dụng third-party pay-out provider để chuyển tiền từ tài khoản của EC sang tài khoản của seller.

Thông thường sẽ sử dụng các third-party account payable provider như Tipalti

## Bước 3: Design Deep Dive

Trong phần này chúng ta sẽ tập trung vào việc làm cho hệ thống:

- Nhanh hơn.
- Bảo mật hơn.
- Đáng tin cậy hơn.

Một vài key topics sau sẽ được xem xét kĩ:

- PSP integration.
- Reconciliation.
- Xử lí payment processing delays.
- Xử lí failed payment.
- Tương tác giữa các internal services.
- Exact-one delivery.
- Tính thống nhất.
- Bảo mật.

### Tích hợp PSP

Trong thực tế thì các payment-systems hiếm khi kết nối trực tiếp tới các ngân hàng hay các card-schemes như Visa hoặc Master-Card, mà thay vào đó sẽ đi theo hướng tích hợp các PSP vào hệ thống của mình theo một trong hai cách như sau:

1. Nếu hệ thống có thể lưu trữ các thông tin nhạy cảm như số tài khoản hay mã số thẻ, ... thì PSP sẽ được tích hợp thông qua API. Hệ thống sẽ chỉ sử dụng PSP để kết nối tới ngân hàng hoặc card-schemes và khi đó `payment web page` sẽ có nhiệm vụ đó là thu thập và lưu trữ các thông tin nhạy cảm về payment.

2. Nếu hệ thống không muốn lưu trữ các thông tin nhạy cảm về payment thì sẽ chọn tích hợp PSP theo hướng sử dụng `hosted payment page` do PSP cung cấp để có thể thu thập các thông tin chi tiết về thẻ thanh toán (việc lưu trữ hoàn toàn do PSP đảm nhận). Đây là cách tiếp cận được nhiều hệ thống triển khai.

![277080470-719b7fce-dd70-470c-8d27-56f0b1acd345](https://github.com/tuananhhedspibk/RoadToSeniorDev/assets/15076665/ff41c7a3-cccc-4106-99e5-ee39925f6533)

Ở hình mô tả về quá trình hosted payment page làm việc ở phía trên, chúng ta sẽ bỏ qua sự hiện diện của `payment-executor` cũng như `ledger` và `wallet` nhằm mục đích đơn giản hoá.

1. User click vào nút "checkout" trên trình duyệt, `payment-service` sẽ được gọi đi kèm với đó là thông tin về `payment-order`.
2. Sau khi nhận được thông tin về `payment-order`, `payment-service` sẽ gửi `payment registration request` sang cho PSP. Request này bao gồm thông tin liên quan đến payment như: lượng tiền, đơn vị tiền, ngày hết hạn của payment request cũng như redirect URL. Do `payment-order` chỉ nên được đăng kí **duy nhất một lần** nên do đó mỗi `payment-order` sẽ có cho mình một trường UUID riêng để đảm bảo tính duy nhất này (UUID còn được gọi là **nonce**), UUID này được dùng luôn làm ID của `payment-order`
3. PSP sẽ trả về 1 payment token, token này sẽ là UUID trên hệ thống của PSP nhằm định danh duy nhất payment registration. Chúng ta sau đó có thể kiểm tra `payment execution status` bằng token này.
4. Payment service sẽ lưu payment token vào DB trước khi gọi đến PSP-hosted payment page.
5. Khi token đã được lưu vào DB, sẽ tiến hành hiển thị PSP-hosted payment page với token, PSP-hosted payment page có khả năng thu thập toàn bộ các thông tin liên quan đến `payment card`, ... và gọi trực tiếp đến hệ thống của PSP, PSP-hosted payment page có thể thu thập các thông tin về card mà không cần phải tương tác với hệ thống của chúng ta. Thông thường PSP-hosted payment page cần 2 loại thông tin như sau:

- Token ta nhận được ở bước 4, PSP code sẽ dùng token này để truy xuất các thông tin về `payment request` từ PSP backend - một thông tin quan trọng ở đây đó là **số tiền sẽ lấy đi**.
- Một thông tin khác đó là **redirect URL**, đây chính là URL sẽ được gọi khi quá trình payment hoàn tất. Khi PSP code hoàn tất quá trình payment thì browser sẽ được chuyển hướng đến **redirect URL** này, thông thường thì **redirect URL** sẽ là URL của trang thông báo trạng thái thanh toán **thuộc về trang EC** (chú ý rằng redirect URL này **hoàn toàn khác** so với webhook ở bước 9).

6. User sẽ nhập các thông tin payment vào PSP web page như credit card number, expiration date, ... và nhấn vào nút pay. PSP sau đó sẽ bắt đầu quá trình thanh toán (payment).
7. PSP trả về payment status
8. Web page lúc này sẽ redirect sang **redirect URL**, payment status nhận được ở bước 7 sẽ được appended vào URL. Ví dụ về full redirect URL: `https://test.com/?tokenId=IJSHDYUW1234&payResult=X324FSa`
9. Một cách bất đồng bộ, PSP sẽ gọi tới `payment service` với thông tin về `payment status` thông qua webhook. `Webhook` này chính là **một URL của payment-service** được đăng kí với PSP ở bước khởi tạo ban đầu. Khi payment-service nhận được payment-events thông qua webhook, nó sẽ bóc tách thông tin về `payment-status` và cập nhật `payment_order_status` trong bảng `Payment Order`.

### Reconciliation

Đây là giải pháp nhằm giải quyết các vấn đề xảy ra trong một payment system khi nó gặp các sự cố như: sự có mạng hoặc có một trong 9 bước như đã trình bày ở phần trên bị failed.

Trong các payment-system thì việc sử dụng asynchronous communication là một điều bình thường nhằm đảm bảo về mặt hiệu năng.

Các external system cũng rất thích asynchronous communication, vậy nên làm cách nào để có thể đảm bảo tính chính xác cho trường hợp này?

Về cơ bản thì **reconciliation** chính là việc "định kì" so sánh trạng thái giữa các services liên quan để từ đó có thể xác nhận rằng hệ thống có đang được vận hành trơn tru hay không. Đây thường là công cụ cuối cùng trong việc bảo vệ một payment system.

Vào mỗi tối, PSP hoặc ngân hàng sẽ gửi cho các clients của họ settlement file, file này bao gồm thông tin về balance của bank account cũng như mọi giao dịch trong ngày của tài khoản đó. Reconciliation system sẽ parse settlement file và so sánh với ledger system.

Hình dưới đây sẽ mô tả cách hoạt động của reconciliation process trong hệ thống.

![Screen Shot 2023-10-21 at 15 53 28](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/9d843d15-b983-4318-b75b-a958f096ba98)

Reconciliation còn có vai trò xác nhận xem các thành phần trong payment system có thống nhất hay không.

Để sửa những lỗi không nhất quán hoặc dữ liệu bị lệch như vậy ta phải chỉnh sửa dữ liệu bằng tay kết hợp với finance team.

Những lỗi gặp phải và việc sửa chúng có thể chia thành 3 nhóm chính như sau:

1. Lỗi có thể phân loại (đã rõ nguyên nhân gây ra lỗi) và việc sửa có thể được tự động hoá. Trong trường hợp này engineer có thể tự động hoá cả việc phân loại lẫn sửa lỗi.
2. Lỗi có thể được phân loại (đã rõ nguyên nhân gây ra lỗi) nhưng ta lại không thể tự động hoá việc sửa được nguyên nhân là do chi phí để tự động hoá việc sửa lỗi là quá cao. Lúc này lỗi sẽ được đưa vào **job queue** và finance team sẽ sửa bằng tay.
3. Lỗi không thể được phân loại (không rõ nguyên nhân gây ra lỗi), đây là trường hợp đặc biệt, cần được đưa vào job queue "đặc biệt", finance team sẽ phải bỏ ra effort để điều tra lỗi và sửa lỗi bằng tay.

### Xử lí payment processing delays

Một payment request được xử lí bởi rất nhiều services con và thường sẽ mất vài giây để hoàn thành, nhưng trên thực tế có những trường hợp độ trễ trong xử lí payment request có thể lên đến cả tiếng đồng hồ, dưới đây là một vài ví dụ:

- PSP nhận thấy payment request có độ rủi ro cao và cần có yếu tố con người trong việc kiểm duyệt nó.
- Credit card cần các phương thức bảo vệ khác như quét 3D bảo mật, nên nó yêu cầu nhiều thông tin chi tiết hơn từ phía card holder.

Payment service cần có khả năng xử lí được các long-running request kiểu này, nếu buy page được host bởi PSP (thường được áp dụng hiện nay) thì PSP sẽ xử lí các long-running payment request theo những cách sau:

- PSP sẽ trả về `pending status` tới client, client sẽ hiển thị trạng thái này cho user, client cũng có thể cung cấp một page để khách hàng kiểm tra trạng thái của payment hiện tại.
- PSP sẽ theo dõi sự thay đổi trạng thái của payment, notify cho payment service thông qua webhook khi có bất kì sự thay đổi nào.

Khi payment request được hoàn thành, PSP sẽ gọi tới webhook của payment-service, payment-service sẽ cập nhật internal state và hoàn tất việc gửi hàng cho khách hàng.

Ngoài cách gọi webhook như trên, một vài PSP sẽ "bắt" payment-service phải "poll" PSP để cập nhật trạng thái của các pending payment requests.

### Tương tác giữa các internal services

Có 2 cách tương tác phổ biến đó là:

- Synchronous
- Asynchronous

#### Synchronous

Tiêu biểu là HTTP, cách làm này hoat động tốt với các hệ thống nhỏ, tuy nhiên khi hệ thống phình to nó sẽ bộc lộ những nhược điểm như sau:

- Hiệu năng thấp: Nếu một trong số các services hoạt động không tốt sẽ làm ảnh hưởng đến toàn bộ hệ thống.
- Dễ tổn thương bởi lỗi: nếu PSP hoặc các services khác failed, client sẽ không thể nhận được response.
- Tăng tính phụ thuộc giữa các services khi phía gửi phải biết về phía nhận.
- Khó để scale: nếu không sử dụng queue như buffer thì khó để hệ thống vận hành được khi traffic tăng đột biến.

#### Asynchronous

Bản thân cách tương tác này cũng được chia thành hai loại:

1. **Single receiver**: Được triển khai với 1 **shared message (request) queue**, message (request) sẽ chỉ được xử lí bởi 1 subscriber hoặc 1 service duy nhất mà thôi, khi message được xử lí nó sẽ được loại bỏ ra khỏi queue như hình minh hoạ sau:

![Screen Shot 2023-10-21 at 16 34 21](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/48f9d079-c47c-4cb2-bdbb-962c43d03419)

2. **Multiple receiver**: Ở đây một request (message) sẽ được xử lí bởi nhiều receivers hoặc services. Kafka sẽ hoạt động rất tốt trong trường hợp này. Cùng một message sẽ được nhận và xử lí bởi nhiều consumers, cách làm này phù hợp hơn trong thực tế cho dù thiết kế cho nó là khá phức tạp. Ta lấy một ví dụ khi một payment event được gửi đến các services khác như:

- Push notification
- Updating financial reporting
- Analytics

có thể minh hoạ như hình vẽ dưới đây:

![Screen Shot 2023-10-21 at 16 54 03](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/5c2f5774-e175-466c-af19-429cde145aff)

`Multiple Receiver` có thể có thiết kế phức tạp hơn `Single Receiver` rất nhiều, xong nó lại phù hợp cho các hệ thống:

- Có business logic phức tạp.
- Quy mô lớn.
- Sử dụng nhiều third-party dependencies.

### Xử lí failed payments

Mọi payment system đều cần phải có cơ chế xử lí lỗi. Tính chịu lỗi và tin cậy chính là chìa khoá ở đây. Dưới đây là một vài kĩ thuật để xử lí failed payment.

#### Theo dõi payment state

Cần định nghĩa rõ ràng vòng đời của một payment để khi có lỗi xảy ra ta có thể dựa theo trạng thái (state) hiện thời để từ đó đưa ra quyết định retry hoặc refund.

Payment state có thể được lưu trữ trong **append-only database**

#### Retry queue và dead letter queue

Để xử lí lỗi, ta có thể triển khai retry queue và dead letter queue theo cách như hình dưới đây:

![Screen Shot 2023-10-21 at 17 29 20](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/3f656be4-36f4-415a-993e-c6d5615b9a17)

Các lỗi không thể retry như ở bước (1b) phía trên thường sẽ được lưu vào DB (VD: invalid input, ...)

Ở bước (3a) ta cần kiểm tra xem số lần retry đã vượt quá ngưỡng định nghĩa trước hay chưa ? Nếu chưa thì sẽ được đưa vào retry queue để retry tiếp, ngược lại sẽ được đưa vào Dead Letter Queue.

- Retry queue: các lỗi có thể retry như các lỗi chỉ mang tính chất tạm thời (transient error)
- Dead letter queue: nếu message bị failed nhiều lần, nó sẽ được đưa vào dead letter queue nhằm mục đích debug sau đó.

### Exact-one delivery

Một trong những vấn đề thường gặp ở payment-system đó là có thể có "double charge" đối với một người dùng.

Chúng ta cần đảm bảo trong thiết kế rằng: ta chỉ charge user **duy nhất một lần** mà thôi.

Có thể tiếp cận vấn đề trên theo hai điều kiện như sau:

1. Hành động được thực thi ít hơn 1 lần (sẽ được trình bày trong phần retry).
2. Hành động được chỉ được thực hiện nhiều nhất 1 lần cùng lúc (sẽ được trình bày trong phần idempotency check).

### Retry

Thông thường ta sẽ tiến hành retry khi gặp sự cố mạng hoặc timeout. Retry sẽ đảm bảo việc request được thực thi ít nhất một lần, như ở hình minh hoạ bên dưới, khi client cố gắng tạo 10$ payment nhưng đều bị failed do sự cố mạng và phải đến lần thư 4 thì mới thành công

![Screen Shot 2023-10-21 at 22 08 27](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/95a0b1c5-0a3a-4f4c-ba1d-9a0114a5ae1d)

Khi tiến hành retry thì khoảng thời gian giữa các lần retries cũng rất quan trọng. Dưới đây là một vài "chiến lược" retry phổ biến:

- Immediate retry: client sẽ retry ngay lập tức.
- Fixed interval: client sẽ retry sau những khoảng thời gian "cố định" được thiết lập từ trước.
- Incremental interval: những lần retry sau sẽ có quãng nghỉ lâu hơn lần retry trước.
- Exponential backoff: retry interval time sẽ tăng theo số mũ. Ví dụ: 1s -> 2s -> 4s -> 8s -> ...
- Cancel: client có thể cancel request, đây là một cách xử lí phổ biến khi payment failed liên tục và "có vẻ" không thể thành công.

Trong thực tế không có một giải pháp nào là "hoàn hảo cả" (one size fits all). Khi tiến hành retry quá nhiều có thể sẽ làm tăng tải cho service. Good practive ở đây đó là cung cấp "errpr-code" đi kèm với **Retry-After** header.

Một vấn đề tiềm tàng khác của retry đó là **double payment**. Cùng xem xét 2 kịch bản sau:

- **Kịch bản 1**: payment service tích hợp với PSP sử dụng **hosted payment page**, user click vào nút pay hai lần.
- **Kịch bản 2**: trong thực tế payment thành công, nhưng do sự cố mạng nên response từ PSP không tới được payment service nên user click nút pay lần thứ hai hoặc phía client retry lại payment.

Để tránh tình trạng "double payment" này, payment chỉ được phép thực hiện **nhiều nhất một lần**. Vấn đề này sẽ do **Idempotency** đảm nhận.

### Idempotency

Từ góc nhìn của API, idempotency được hiểu là có thể tạo **nhiều requests** nhưng chỉ trả ra **một kết quả duy nhất** mà thôi.

Khi tiến hành tương tác với server, client sẽ tạo ra một key (có expire time), key này thường sẽ là UUID, idempotent key sẽ được thêm vào HTTP header: `<idempotency-key: key_value>`.

Sau đây chúng ta sẽ cùng tìm hiểu việc idempotent key giải quyết 2 kịch bản ở phần trên như thế nào.

#### Kịch bản 1: Cần làm gì khi user click nút "pay" hai lần?

Khi user click vào nút pay, idempotent key sẽ được gửi đến server (với trang EC thì có thể là cart ID trước khi checkout), request thứ 2 khi đến server sẽ được server nhìn nhận là retry do trước đó server đã thấy idempotent key này rồi nên server sẽ trả về trạng thái cuối cùng của request trước.

![Screen Shot 2023-10-21 at 22 29 51](https://github.com/tuananhhedspibk/micro-buying/assets/15076665/de81ec49-ff83-4b26-9b31-8dda315d19c8)

Nếu có nhiều request khác đến với cùng idempotent-key thì chỉ có request đầu được xử lí, các request sau sẽ được trả về **429 status code - Too Many Request**.

Để thực hiện idempotent-key, ta có thể sủ dụng DB unique key constraint. VD: ID của DB sẽ được sử dụng như idempotent-key (ở đây ID là primary key).

1. Khi payment system nhận payment, nó sẽ insert vào DB.
2. Nếu insert thành công chứng tỏ payment request này là request đầu tiên.
3. Nếu insert thất bại do **primary key đã tồn tại trước đó** thì chứng tỏ request này là request đến sau và sẽ không được xử lí.

#### Kịch bản 2: Payment thành công nhưng response của PSP không đến được payment system do sự cố mạng, user nhấn nút "pay" lần thứ hai

Như đã trình bày ở các phần trước, payment service sẽ gửi đến PSP một **nonce** và PSP sẽ trả về **token**. Cả **nonce** và **token** này đều tương ứng với một payment order duy nhất. Khi user nhấn "pay" lần thứ hai, payment order vẫn thế nên **token** vẫn thế và vì **token** được PSP sử dụng như một **idempontency key** nên PSP có khả năng nhận ra rằng đây là "double payment" và trả về trạng thái của lần thực thi trước.

### Consistency

Dữ liệu được lưu ở khá nhiều nơi khi quá trình payment diễn ra.

1. Payment service lưu token, nonce, ...
2. Ledger lưu accounting data.
3. Wallet lưu account balance của merchant
4. PSP lưu payment execution status
5. Dữ liệu được replicated giữa các databases.

Khi liên lạc giữa 2 services gặp sự cố sẽ dẫn đến tình trạng dữ liệu bị bất đồng bộ, do đó để đảm bảo sự đồng bộ về mặt dữ liệu giữa các services, ta cần đảm bảo **exactly-once processing**

Ngoài ra ta có thể sử dụng idempotent key cũng như reconciliation (do không phải lúc nào external service cũng đúng)

### Payment security

Vấn đề bảo mật cũng cần phải được xem xét một cách nghiêm túc ở đây.

Ta có thể sử dụng:

- HTTPs để mã hoá request
- SSL để tránh man-in-the-middle attack
- Rate-limitting và firewall để tránh DDos.
- Sử dụng token thay vì card number.

## Bước 4: Tổng kết

Ngoài việc thiết kế pay-in và pay-out flow. Ta cũng cần chú ý đến:

- Monitoring: để biết CPU, memory sử dụng.
- Alerting: dev có thể xử lí kịp thời các vấn đề của hệ thống.
- Debug tool.
- Đổi đơn vị tiền tệ.
- Địa lí: mỗi khu vự sẽ có các payment methods khác nhau.
- Cash payment
- Google/ Apple pay integration.
