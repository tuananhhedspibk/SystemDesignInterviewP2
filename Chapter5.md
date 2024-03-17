# Chương 5: Metrics monitoring và Alerting system

Việc có một hệ thống monitoring và alert tốt sẽ giúp hệ thống đảm bảo:

- Tính sẵn có cao
- Tính tin cậy cao

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

Tuỳ vào từng công ty mà khái niệm monitoring cũng như alert là khác nhau, nên do đó việc làm rõ requirement là rất quan trọng.

**Yêu cầu**:

- Build internal logging system.
- Metrics: OS (**low-level** usage data: CPU load, memory usage, disk space consumption, **high-level**: requests per second).
- DAU: 100 triệu, 1000 server pools, 100 servers mỗi pool.
- Chúng ta sẽ lưu trữ dữ liệu metrics trong 1 năm.
- Về "độ phân giải - resolution": Nhận mới dữ liệu trong vòng 7 ngày, sau 7 ngày có thể đưa lên cỡ "1 phút" trong 30 ngày, sau 30 ngày có thể là 1 giờ.
- Hỗ trợ các alert channels: Email, Phone, PagerDuty, webhook.
- Không cần lấy về error logs hoặc access logs.
- Không cần hỗ trợ Distributed System tracing.

**Non-functional requirements**:

- Scalability: hệ thống nên có khả năng scale khi số lượng metrics và alert volume tăng lên.
- Độ trễ thấp
- Tính tin cậy cao nhằm tránh việc mất đi các critical alerts.
- Dễ dàng tích hợp các kĩ thuật mới sau này.

## Bước 2: High-level design

### Cơ sở

Metric monitoring và alerting system thường yêu cầu 5 components như sau:

1. **Data Collection**: lấy về các metric data từ các nguồn khác nhau.
2. **Data transmission**: chuyển dữ liệu từ sources sang metrics monitoring system.
3. **Data storage**: tổ chức và lưu các dữ liệu đến - incoming data.
4. **Alerting**: phân tích dữ liệu đến, tìm ra điểm bất thường, generate alerts. Hệ thống cần có khả năng gửi alerts đến các communication channels khác nhau.
5. **Visulization**: biểu diễn dữ liệu ở dạng đồ thị, ... để enigeers có thể tìm ra các pattern cũng như trends, ...

### Data model

Metrics data thường được ghi lại dưới dạng **time series** bao gồm (value - time).

VD1:

![Screenshot 2024-03-17 at 23 27 18](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/f63d5921-c7b5-450c-90a5-e98b1cdfe8df)

VD2:

CPU load trung bình trên toàn bộ các web servers thuộc region `us-west` trong 10 phút trước ? Về mặt lý thuyết chúng ta có thể "kéo về" các thông tin như phía dưới với metric name là `CPU.load` và region là `us-west`.

```txt
CPU.load host=webserver01,region=us-west 1613707265 50
CPU.load host=webserver01,region=us-west 1613707265 62
CPU.load host=webserver02,region=us-west 1613707265 43
CPU.load host=webserver02,region=us-west 1613707265 64
...

CPU.load host=webserver01,region=us-west 1613707265 76
CPU.load host=webserver01,region=us-west 1613707265 83
```

Lượng CPU load trung bình có thể tính bằng phép tính trung bình các giá trị cuối ở mỗi dòng trên. Format của mỗi dòng trong ví dụ này được gọi là `line protocol`.

Mọi dữ liệu time series đều bao gồm những yếu tố sau:

![Screenshot 2024-03-18 at 7 34 45](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a4536ab5-8b58-4fc9-b963-8e935e452dac)

### Data access pattern

Trong hình dưới đây, trục y biểu diễn time-series trong khi trục x biểu diễn time.

![Screenshot 2024-03-18 at 7 39 04](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/859db3d1-2fd9-4ce3-9e90-0dbac5c4d7f3)

Write load khá nặng, như đã đề cập trong phần "High-level requirements", khoảng 10 triệu operational metrics được ghi mỗi ngày nên traffic khá nặng về việc ghi (write-heavy).

Read load có thể nhìn nhận như một dạng đồ thị hình sin (lúc nhiều, lúc ít), cả visualization và alerting services đều gửi queries đến DB, tuỳ thuộc vào access patterns của graphs và alerts mà read volume có thể tăng đột biến.

### Data storage system
