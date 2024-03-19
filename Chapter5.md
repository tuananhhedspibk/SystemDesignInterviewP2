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

Data storage system chính là trái tim của thiết kế, chúng ta không nhất thiết phải xây dựng một storage system của riêng mình cũng như sử dụng các storage system phổ biến như MySQL.

Sử dụng các DB phổ biến có thể lưu trữ time-series data nhưng nó yêu cầu một **expert-level tuning** cho việc đáp ứng quy mô (scale) của hệ thống.

RDB không hề tối ưu cho việc thực thi các thao tác vận hành với time-series data. VD như việc tính "bước di chuyển trung bình - moving average" trong một "rolling time window" yêu cầu các SQL phức tạp.

Hơn nữa, với tagging hoặc labeling data yêu cầu ta cần đánh index cho từng tag.

Với heavy write thì các RDB phổ biến không đáp ứng tốt về mặt hiệu năng.

Còn với NoSQL, ta có thể kể ra một vài công cụ như Cassandra, Bigtable - như những công cụ phổ biến cho việc xử lí time-series data. Thế nhưng để tối ưu cho việc query các time-series data, cần phải xây dựng một scalable schema cho chúng.

Có rất nhiều các storage system hỗ trợ:

- Tối ưu cho time-series data.
- Sử dụng ít server hơn cho việc xử lí cùng một lượng dữ liệu.
- Custom query interface được thiết kế riêng cho việc phân tích time-series data (dễ hơn SQL rất nhiều).
- Hỗ trợ data retention và data aggregation.

Một vài ví dụ về time-series DB như sau:

`OpenTSDB` - nó dựa trên Hadoop và HBase hoặc `Timestream` của Amazon. Ngoài ra còn hai time-series DB nổi tiếng là `InfluxDB` và `Prometheus`.

Về cơ bản metric-data sẽ có dạng time-series. Một tính năng mạnh khác của time-series DB đó là khả năng:

- Kết tập dữ liệu
- Phân tích một lượng lớn time-series data bằng labels (hoặc tags)

VD: InfluxDB xây dựng index trên label với mục đích tăng khả năng tìm kiếm trên time-series data bằng labels nhanh hơn, đây là một tính năng rất quan trọng đối với việc visualization mà thực sự rất khó để xây dựng với các DB thông thường khác.

### High-level design

![Screenshot 2024-03-19 at 23 52 45](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/9ad49c2e-18ce-41bf-9416-dd467d773e2f)

- **Metrics source**. Đây có thể là application server, SQL database, message queue.
- **Metrics collector**. "Thu gom metrics data" và ghi nó lên time-series DB.
- **Time-series database**. Lưu metrics data dưới dạng time-series. Cung cấp các custom query interfaces cho mục đích phân tích và kết tập một lượng lớn dữ liệu dưới dạng time-series. Ngoài ra nó còn maintaince indexes trên labels để tăng tốc độ tìm kiếm bằng labels.
- **Query service**. Giúp việc query dữ liệu từ Time-series DB trở nên dễ dàng hơn, service này cũng có thể được thay thế bởi query interface của bản thân time-series DB.
- **Alerting system**. Gửi alert notification đến các alerting destinations.
- **Visualization System**. Show metrics dưới các dạng graph/charts khác nhau.

## Bước 3: Deep-dive design

Trong phần này chúng ta sẽ tập trung vào:

- Metrics collection
- Scaling metrics transmission pipeline
- Query service
- Storage layer
- Alerting system
- Visualization system

### Metrics collection

![Screenshot 2024-03-19 at 23 53 08](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/37d4602d-1091-41c3-856b-f4d4eddd004c)

#### Pull vs push models

Có 2 cách để thu thập metrics data đó là **pull** và **push**, mỗi một cách làm sẽ có những ưu điểm của riêng nó.

**Pull model**

![Screenshot 2024-03-19 at 23 57 03](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/692437fa-2397-4da8-8fa6-097ef3ce428b)

Trong cách tiếp cận này, metrics collector cần phải biết danh sách các end points để kéo dữ liệu về. Có một cách làm khá "ngây ngô" đó là lưu file chứa thông tin về DNS/IP của các service endpoint trên "metrics collector" servers. Cách làm này khá đơn giản nhưng sẽ khó để maintaince khi quy mô hệ thống trở nên lớn hơn. Thực tế là khi số lượng server tăng, giảm hoặc bị thay thế xảy ra là khá thường xuyên.

Thế nhưng trong thực tế có những `Service Discovery` với độ tin cậy cao như `ZooKeeper` hay `etcd` sẽ là giải pháp phù hợp cho chúng ta.

Service Discovery bao gồm các configuration rules về việc khi nào sẽ thu tập metrics data và thu thập ở đâu.

![Screenshot 2024-03-19 at 23 57 47](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/da490c21-cb2f-4369-8345-b82535fa0e0b)

Hình dưới đây là minh hoạ cho pull model một cách chi tiết hơn.

![Screenshot 2024-03-19 at 23 59 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/8ae976f4-9250-4b28-a076-9aaf3cd7078f)

1. Metric collector kéo các configure metadata về service endpoints về từ Service Discovery. Metadata này sẽ bao gồm 　 **pulling interval**, **IP addresses**, **timeout**, **retry parameters**.
2. Metric collector pull metrics data thông qua một pre-defined HTTP endpoint.
3. Optional, metrics collector có thể đăng kí `change event notification` với Service Discovery để nhận các thông báo khi service endpoints có sự thay đổi. Ngoài ra, metrics collector có thể poll endpoint changes một cách định kì.

Ở quy mô của chúng ta, một metrics collector sẽ không có khả năng xử lí cả nghìn servers, thay vào đó chúng ta bắt buộc phải sử dụng metrics collector poll để xử lí.lí

Một vấn đề khác khi có nhiều metrics collector đó là việc chúng sẽ cùng pull dữ liệu từ một nguồn và tạo ra các metrics data giống hệt nhau. Do đó cần có một cơ chế điều phối giữa các instances để tránh điều đó.đó

Một giải pháp khá hay ở đây đó là đưa mỗi collector vào một khoảng (range) trong consistent hash ring sau đó map các server với các collector bằng tên của chúng trong hash ring. Điều này đảm bảo **một metrics source server sẽ được xử lí bởi duy nhất một collector**.

![Screenshot 2024-03-20 at 8 32 26](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a209d177-1247-4dab-828c-8090fa0cdc43)

Như ở hình trên chúng ta thấy collector 2 sẽ đảm nhận 2 servers đó là `S1` và `S5`
