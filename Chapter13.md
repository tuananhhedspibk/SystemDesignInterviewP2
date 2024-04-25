# Chương 13: Stock Exchange (Trao đổi chứng khoán)

Chức năng cơ bản của một hệ thống exchange đó là matching giữa buyers và sellers. Trước thời đại của máy tính, con người thường trao đổi bằng hàng hoá và bằng lời nói.

Không những trao đổi hàng hoá, ngày nay con người còn trao đổi cả những suy luận cũng như kinh doanh chênh lệch giá.

Công nghệ đang thay đổi mạnh mẽ lĩnh vực marketing cũng như đầu tư, thương mại.

Khi nhắc đến trao đổi chứng khoán, thông thường chúng ta sẽ nghĩ đến sàn chứng khoán New York hoặc Nasdaq, ... thế nhưng có rất nhiều loại trao đổi, một vài trong số đó tập trung vào chiều dọc của ngành công nghiệp tài chính.

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

- Hệ thống chỉ tập trung vào stock trading.
- Hệ thống cần hỗ trợ: đặt lệnh mới (new order), cancel lệnh (order canceling), với order type, hệ thống chỉ cần hỗ trợ limit order (hạn chế đặt lệnh).
- Hệ thống chỉ cần hỗ trợ trading hours thông thường.
- Client có thể đặt một số lệnh nhất định, cancel chúng cũng như nhận về các kết quả trading real-time. Client có thể xem được real-time order book (danh sách lệnh mua và bán). Cần hỗ trợ ít nhất 10,000 users trading tại cùng một thời điểm. Về trading volume, chúng ta cần hỗ trợ cỡ "tỉ" orders mỗi ngày. Cần đảm bảo có risk check.
- Về risk check ta có thể ví dụ đơn giản như sau: user chỉ có thể mua tối đa 1 triệu Apple stock trong một ngày.
- Chúng ta cũng cần đảm bảo rằng users có đủ lượng tiền để có thể đặt lệnh.

  ### Non-functional requirements

  - **Tính sẵn có**
