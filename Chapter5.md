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

##### Pull model

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

##### Push model

Như hình dưới đây ta thấy các metrics source sẽ gửi metrics data đến metrics collector.

![Screenshot 2024-03-20 at 10 00 40](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a078dc22-8263-4a41-8eca-7d97e1106a11)

Trong push model, `collection agent` thường được cài đặt trong các server đang được monitored, các agents này có thể là `long-running software` - với nhiệm vụ thu thập metrics từ service chạy trên servers và gửi nó cho các collectors một cách định kì.

Các collection agent có thể "kết tập" các metrics dưới local cuả mình trước khi gửi chúng cho metric collectors.

"Kết tập - aggregation" là một cách hữu hiệu trong việc giảm đi lượng dữ liệu gửi cho metrics collector.

Trong trường hợp push traffic cao và metrics collector reject push, agent sẽ giữ một buffer data nhỏ dưới local của mình (thường là lưu trong ổ đĩa) và gửi lại cho collector sau đó. Tuy nhiên trong trường hợp agent thuộc về một server nằm trong auto-scaling group (sẽ đến một lúc nào đó server này sẽ bị kill). Khi server bị kill thì dữ liệu buffer dưới local của nó cũng sẽ mất đi và metrics collector sẽ bị chậm đi về mặt dữ liệu.

Để tránh tình trạng metrics collector bị chậm về mặt dữ liệu, metrics collector nên được đặt trong một auto-scaling cluster với load balancer ở phía trước nó như hình dưới đây. Việc scale up hay scale down của cluster sẽ dựa theo CPU load của metric collector server.

![Screenshot 2024-03-20 at 10 13 24](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/4c068290-4ff1-4c84-bb28-702092f12cdb)

**Pull hay push ?**

Vậy thì mô hình nào sẽ phù hợp hơn cho chúng ta, trên thực tế sẽ là không có một mô hình nào là tối ưu hoàn toàn cả, tất cả sẽ phải phụ thuộc vào tình huống mà chúng ta gặp phải.

- Ví dụ về pull architecture bao gồm Prometheus.
- Ví dụ về push architecture bao gồm Amazon Cloudwatch và Graphite.

> Biết được ưu, nhược điểm của mỗi cách tiếp cận quan trọng hơn rất nhiều so với việc tìm ra mô hình "chiến thắng"

Dưới đây là một vài ưu nhược điểm cần cân nhắc giữa các models.

- Dễ dàng debug: **Pull** win
- Health check: **Pull** win

#### Scale metrics transmission pipeline

![Screenshot 2024-03-20 at 22 42 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e02acec3-b4ea-475a-836f-d0c59ca04e99)

Chúng ta cùng thử đào sâu vào `metrics collector` và `time-series DB`. Dù là push hay pull model thì metrics collector là một cluster servers, cluster này sẽ nhận về một số lượng lớn dữ liệu. Dù là push hay pull model thì ta cũng cần phải thiết lập auto-scaling để đảm bảo có đủ collector instances.

Thế nhưng rủi ro bị mất dữ liệu nếu time-series database không hoạt động là hoàn toàn có thể xảy ra. Để giải quyết vấn đề này chúng ta sẽ sử dụng `queueing component` như hình dưới đây.

![Screenshot 2024-03-21 at 8 11 06](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/186c86f7-9fab-4eaa-9ab6-77faa73438bf)

Trong thiết kế này, metrics collector sẽ gửi metrics data đến queueing system như Kafka. Consumers hoặc streaming processing services như Âpche Storm, Flink hay Spark sau đó sẽ xử lí và push dữ liệu đến time-series database. Cách tiếp cận này có một vài ưu điểm như sau:

- Kafka được sử dụng như một distributed messaging platform với độ tin cậy và khả năng mở rộng cao.
- Giúp decouple data collection và data processing services.
- Do dữ liệu được lưu tạm thời ở Kafka nên tránh được tình trạng mất dữ liệu khi database không hoạt động.

#### Scale thông qua Kafka

Có một vài cách mà ở đó chúng ta có thể tận dụng cơ chế phân vùng (partition) sẵn có của Kafka để scale hệ thống của chúng ta.

- Thiết lập số lượng partitions dựa theo yêu cầu về thông lượng.
- Phân vùng metrics data theo metrics names để từ đó consumers có thể kết tập dữ liệu theo metrics names.
- Phân vùng metrics data sâu hơn nữa theo tags/labels.
- Phân loại (categorize) và phân cấp độ ưu tiên (prioritize) metrics để có thể xử lí các metrics quan trọng trước.

![Screenshot 2024-03-21 at 8 22 07](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/8c4d9e97-35a3-4cf9-8289-7e4318c0499b)

##### Thay thế Kafka

Maintaince một hệ thống như Kafka trong môi trường product là một thách thức không hề nhỏ. Có những monitoring system lớn không hề sử dụng intermediate queue, ví dụ như Facebook Gorilla in-memory time-series DB là một ví dụ điển hình, nó được thiết kế để đảm bảo việc tính sẵn có cao cho thao tác ghi. Đây có thể sẽ là một cuộc tranh luận về tính tin cậy với các hệ thống sử dụng intermediate queue như Kafka.

##### Những nơi mà Aggregation có thể xảy ra

Aggregation có thể xảy ra ở nhiều nơi khác nhau:

- Tại collection agent (client-side)
- Ingestion pipeline
- Query side

  **Collection agent**. Collection agent được cài đặt phía client chỉ hỗ trợ các aggregation logic đơn giản. Ví dụ: aggregate counter mỗi phút trước khi gửi cho metrics collector.

  **Ingestion pipeline**. Để kết tập dữ liệu trước khi ghi vào storage, ta thường cần một streaming process engine như Flink. Write volume sẽ chỉ giảm đi chỉ khi ta lưu dữ liệu đã tính toán vào trong DB. Và chúng ta sẽ mất đi tính "khả chuyển" của dữ liệu vì chúng ta không còn lưu raw data.

  **Query Side**. Raw data có thể được kết tập theo từng khoảng thời gian nhất định. Sẽ không có data loss với cách tiếp cận này nhưng tốc độ query có thể sẽ bị chậm đi vì kết quả query sẽ được tính ở thời điểm query và chạy lại với toàn bộ dataset.

#### Query Service

Query service bao gồm cluster của các query servers, các servers này sẽ nhận requests từ visualization hoặc alerting system 　 và query trực tiếp đến time-series DB. Query service sẽ giúp decouple time-series DB với client (visualization hoặc alerting system).

Việc này giúp ta có thể dễ dàng thay đổi time-series DB hoặc visualization, alerting system nếu cần thiết.

##### Cache layer

Để giảm tải cho time-series DB và query service, chúng ta sẽ thêm cache servers để lưu query results như hình dưới đây.

![Screenshot 2024-03-21 at 22 30 25](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/2a9d0889-9fbf-4b59-b9df-f1ee2ac72c3a)

##### Trường hợp với query service

Trên thực tế sẽ có những time-series DB hỗ trợ các query interface mạnh nên cũng không nhất thiết phải thêm query service vào hệ thống.

##### Time-series database query language

Các metrics monitoring system như Prometheus hay InfluxDB không sử dụng SQL mà có query language của riêng mình. Một lí do chính ở đây đó là việc xây dựng SQL query cho time-series data là không hề đơn giản. Ta lấy ví dụ với việc tính trung bình hàm mũ bằng SQL sẽ như sau:

![IMG_2725](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/590dba0c-57bb-4077-bb2b-ca1fbc48acf6)

Trong khi đó với Flux, một ngôn ngữ được tối ưu hoá cho time-series analysis (được sử dụng trong InfluxDB), việc tính toán sẽ như sau:

```js
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

#### Storage layer

##### Chọn time-series database một cách cẩn thận

Theo như nghiên cứu của Facebook, ít nhất 85% tổng số queries tới `operational data store` là để thu thập dữ liệu trong vòng 26 giờ vừa qua. Nếu chúng ta sử dụng một time-series database đáp ứng được điều kiện trên, hiệu năng của toàn bộ hệ thống sẽ được cải thiện một cách đáng kể.

##### Tối ưu hoá không gian nhớ

Do lượng metric data được lưu là rất lớn, dưới đây là một vài giải pháp đối phó cho vấn đề này.

**Mã hoá và nén dữ liệu:**

Việc mã hoá và nén giúp giảm đi kích thước của dữ liệu. Dưới đây là một ví dụ đơn giản:

![Screenshot 2024-03-21 at 23 17 41](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e17bb4ef-5081-4982-9cab-792e7835ea46)

Như ở ví dụ trên, giá trị `1610087371` và `1610087381` chỉ khác nhau có 10s (chỉ cần 4 bit để biểu diễn sự khác nhau này), thay vì biểu diễn full timestamp với 32 bits, ta chỉ cần lưu delta giữa các giá trị là đủ. Do đó, tập giá trị cần lưu sẽ bao gồm:

- Giá trị cơ bản nhỏ nhất
- Các giá trị delta

Cụ thể là: `1610087371`, `10`, `10`, `9`, `11`.

##### Downsampling

Là quá trình convert dữ liệu từ độ phân giải cao xuống độ phân giải thấp để giảm đi dung lượng bộ nhớ sử dụng. Do dữ liệu của chúng ta chỉ có "hạn sử dụng" là 1 năm nên chúng ta có thể downsample những dữ liệu cũ. Ví dụ: chúng ta có thể để cho engineer và data scientist đưa ra các nguyên tắc khác nhau cho các metrics khác nhau như sau:

- Retention: 7 ngày, không sampling.
- Retention: 30 ngày, downsample xuống `1 min resolution`.
- Retention: 1 năm, downsample xuống `1 hour resolution`.

Cùng lấy một ví dụ cụ thể khác, khi chúng ta kết tập dữ liệu theo độ phân giải 10-giây, chúng ta sẽ có:

![Screenshot 2024-03-22 at 7 32 08](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/8ced7600-c7d1-4aa0-8b95-6a09830622c9)

Nếu kết tập với độ phân giải 30-giây, chúng ta sẽ có:

![Screenshot 2024-03-22 at 7 33 00](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ef5fa5c3-b6bd-41ad-aa70-1f6243738f96)

##### Cold storage

Cold storage được sử dụng cho các `inactive data`, giá thành cho cold storage cũng khá thấp.

#### Alerting system

![Screenshot 2024-03-22 at 7 39 36](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/a6db8f39-dd08-4fba-9482-c90d7aab4958)

Alert work flow sẽ hoạt động như sau:

1. Load config files vào cache servers. Các rules sẽ được định nghĩa trong file config này (thường dưới YAML format). Ví dụ:

```YAML
- name: instance_down
rules:

# Alert for any instances that is unreachable for >5 minutes
- alert: instance_down
    expr: up == 0
    for: 5m
    labels:
    severity: page
```

2. Alert Manager fetch config từ cache.
3. Dựa theo config rules, alert manager sẽ gọi query service theo những khoảng thời gian định trước. Nếu giá trị vượt quá một threshold được định nghĩa trước, alert event sẽ được tạo. Alert manager có trách nhiệm thực hiện những việc sau:

- Filter, merge, dedupe alerts. Dưới đây là ví dụ về việc merge alerts được trigger bên trong một instance trong một khoảng thời gian ngắn.

![Screenshot 2024-03-22 at 7 57 41](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/7575bbf6-617a-4937-ac4c-ae83436e77b0)

- Access control. Để đảm bảo vấn đề về bảo mật cũng nhưng tránh các human error, ta nên chỉ cho phép một vài người có thẩm quyền nhất định truy cập vào alert management operations.
- Retry. Alert manager kiểm tra alert state và đảm bảo rằng notification sẽ được gửi đi ít nhất là 1 lần.

4. Aler store là một key-value DB như là Cassandra, sẽ lưu state (inactive, pending, firing, resolved) của mọi alerts. Nó đảm bảo notification sẽ được gửi ít nhất một lần.
5. Insert alert vào kafka.
6. Alert consumers pull alert events từ kafka.
7. Alert consumers xử lí alert events từ Kafka và gửi notifications đến các channels như email, text message, PagerDuty hoặc HTTP endpoints.

##### Alerting system - build vs buy

Trong thực tế có nhiều alerting system có khả năng tích hợp tốt với các time-series DB cũng như các notification channel như email hay PagerDuty. Trong quá trình phỏng vấn cho vị trí senior việc buy hay mua hệ thống có sẵn hoàn toàn phụ thuộc vào quyết định của bạn.

##### Visualization system

Được built phía trên cùng của data layer. Các metrics có thể được show trên dashboard.

<img>

Việc tự mình xây dựng một visualization system không phải là một điều dễ dàng, hiện có rất nhiều hệ thống có sẵn như Grafana có khả năng tich hợp tốt với các time-series DB phổ biến.

## Bước 4 - Tổng kết

Trong phần này chúng ta đã nói về:

- Pull vs push model.
- Utiliza Kafka để scale hệ thống.
- Chọn đúng time-series DB.
- Sử dụng downsampling để giảm data size.
- Build vs buy options cho alerting và visualizing system.

  Đây chính là thiết kế cuối cùng của chúng ta.

  <img>
