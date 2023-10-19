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

### Back-of-the-envelopre estimation

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
