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

- **Tính sẵn có**: ít nhất 99.99% phải đảm bảo. Nếu có downtime vài giây, sẽ gây ra những ảnh hưởng lớn.
- **Khả năng chịu lỗi**: Khả năng chịu lỗi và cơ chế phục hồi nhanh là cần thiết để hạn chế tối đa ảnh hưởng của product incident.
- **Độ trễ**: round-trip latency chỉ nên ở mức mili giây. Round trip latency được đo từ thời điểm `market order một exchange` cho tới thời điểm `market order trả về một lần thực thi thành công`.
- **Bảo mật**: hệ thống nên có account management, đối với ... hệ thống sẽ thực thi KYC (Know Your Client) check để kiểm chứng user identity trước khi một account mới được mở. Đối với public resources như các trang web bao gồm market data, chúng ta nên phòng tránh DDoS attack.

### Back-of-the-envelope estimation

Hệ thống có:có

- 100 symbols
- 1 tỉ orders mỗi ngày
- NYSE Stock exchange mở từ thứ 2 đến thứ 6 (9.30am - 4.00pm). Mỗi ngày tổng cộng 6.5 giờ
- QPS = `1 tỉ / (6,5 x 3,600) =~ 43,000`
- Peak QPS = `5 x QPS = 215,000`. Lượng giao dịch sẽ cao khi thị trường mở vào buổi sáng và trước khi khép lại vào buổi chiều.

## Bước 2: High-level design

### Business knowledge 1010

#### Broker

Hầu hết các clients bán lẻ đều thực hiện giao dịch thông qua broker. Một vài broker có thể bạn sẽ quen thộc bao gồm Charles Schwab, Robinhood.

Các brokers này cung cấp giao diện thân thiện giúp các nhà đầu tư đơn lẻ đặt lệnh vào xem dữ liệu của thị trường.

#### Institutional client

Institutional client sẽ sử dụng phần mềm đầu tư đặc biệt khi số lượng giao dịch đầu tư của họ rất lớn. Các Institutional clients khác nhau sẽ có các yêu cầu khác nhau. VD: quỹ hưu trí hướng tới thu nhập ổn định, chúng thường ít khi được đầu tư nhưng khi đầu tư, khoản đó sẽ rất lớn. Khoản đầu tư này cần phải được chia nhỏ để giảm ảnh hưởng tới thị trường chung.

#### Limit order

Limit order là một lệnh mua hoặc bán với giá cố định. Nó có thể không thấy khớp lệnh ngay lập tức nhưng có thể nó sẽ khớp một phần.

#### Market order

Market order không chỉ định giá. Chúng được thực thi dựa theo giá hiện hành trên thị trường. Market order hi sinh giá để đảm bảo sực thực thi. Chúng khá hữu dụng trong một vài thị trường mua bán nhanh.

#### Market data levels

Thị trường chứng khoán Mỹ có 3 quotes giá: L1 (level 1), L2 và L3. L1 market data bao gồm bid price và ask price tốt nhất cũng như số lượng

![Screenshot 2024-05-11 at 21 30 13](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/35e7f43a-fc77-48a6-9eb6-b828c3be013f)

L2 sẽ bao gồm nhiều mức giá hơn L1

![Screenshot 2024-05-11 at 21 42 16](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/8b572b0f-0b27-4db9-b0a7-65cc3d06a03a)

L3 cho thấy các mức giá và queued quantity ở mỗi mức giá

![Screenshot 2024-05-11 at 21 42 20](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/5326977a-1223-4c2c-b611-5bc9eef44671)

#### Candlestick chart (biểu đồ nến)

Biểu đồ nên hiển thị giá cổ phiếu theo từng khoảng thời gian. Một biểu đồ nến điển hình trông sẽ như sau:

![Screenshot 2024-05-11 at 21 47 03](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/2cec5706-2a93-4f7a-ae3e-d57bea11c98e)
