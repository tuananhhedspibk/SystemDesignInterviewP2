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

- Pay-in flow:
