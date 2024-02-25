# Digital Wallet

Các Payment Platform thường cung cấp digital wallet service cho client để từ đó client có thể nạp tiền vào cũng như thanh toán thông qua digital wallet

![Screenshot 2024-02-12 at 22 43 38](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/0cac8bee-4eac-4a3a-872b-1c991fef1b2c)

Digital wallet còn cho phép chuyển tiền giữa các wallet với nhau, việc chuyển tiền giữa các wallet với nhau sẽ nhanh hơn bank-to-bank rất nhiều hơn nữa còn không mất thêm phụ phí.

![Screenshot 2024-02-12 at 23 05 45](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/92c00288-da63-415b-bea9-01e6e6afed84)

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

- Chỉ tập trung vào tính năng chuyển tiền giữa 2 digital wallets
- 1,000,000 TPS (Transaction Per Second)
- Kiểm tra tính chính xác của các giao dịch: xác nhận qua dữ liệu của các transactions cũng như statements từ ngân hàng, một nhược điểm của quá trình reconciliation đó là nó chỉ đưa ra sự "dị thường" về mặt dữ liệu chứ không đưa ra được nguyên nhân tại sao phát sinh sự "dị thường" đó, vì thế chúng ta cần thiết kế một hệ thống với khả năng "reproducibility" tức là chúng ta có khả năng "tái tạo lại" lịch sử giao dịch dựa theo dữ liệu sẵn có.
- Reliability ít nhất là 99.99%

### Back-of-the-envelope estimation

Khi nói về khá niệm TPS chúng ta sẽ ngầm hiểu đó là transaction database sẽ được sử dụng.

Ở thời điểm hiện tại một data center node có thể hỗ trợ "vài nghìn TPS" - ta lấy ví dụ với data center node hỗ trợ 1000 TPS, với yêu cầu hệ thống (1,000,000 TPS) thì ta cần 1000 database nodes.

Trên thực tế một giao dịch sẽ bao gồm 2 quá trình:

- Lấy tiền đi từ 1 account
- Chuyển tiến đến 1 account

Do đó với 1 giao dịch, chúng ta cần 2 TPS, do đó tổng số node cần có là 2000 nodes

## Bước 2: High level design

### API design

Chúng ta sẽ sử dụng RESTful API convention. Ở đây ta chỉ cần 1 API duy nhất

`POST /v1/wallet/balance_transfer` - chuyển tiền từ 1 wallet sang wallet khác

Các request params sẽ như sau:

`from_account`: debit account - string

`to_account`: credit account - string

`amount`: lượng tiền chuyển - string

`currency`: đơn vị tiền tệ - string

`transaction_id`: ID sử dụng để tránh duplicate (deduplication) - uuid

Sample response body:

```JSON
{
  "Status": "success",
  "Transaction_id": "0152746273-23723-11ec-1232137-3489138ac13213"
}
```

Về `amount` - vẫn có một số trường hợp sử dụng float hoặc double number, tuy nhiên nên nắm rõ về risk trước khi sử dụng nó.

### In-memory sharding solution

Ta có thể sử dụng cấu trúc map (hash map) `<user, balance>` để lưu số dư tài khoản cho từng user.

Với in-memory stores, một giải pháp phổ biến ở đây đó là `Redis`, 1 Redis node là không đủ để xử lí 1 triệu TPS, do đó ta cần một cluster các Redis nodes và điều phối các transactions giữa chúng - quá trình này gọi là `partitioning` hoặc `sharding`

Để phân phối key-value data giữa `n` partitions, ta có thể tính hash value của key và chia nó cho `n`, số dư thu được chính là index của partition mà key-value được phân vào.

Đoạn mã giả dưới đây mô phỏng lại quá trình phân vùng dữ liệu.

```ts
const accountId = 'A';
const partitionNumber = 7;
const myPartition = accountId.hashCode() % partitionNumber;
```

Số lượng partitions và việc định danh các Redis nodes có thể được lưu trữ và đảm nhận bởi ZooKeeper - centralized place

Thành phần cuối cùng ở đây đó là một service xử lí các transfer commands - wallet service, nó có các nhiệm vụ chính như sau:

1. Nhận transfer command
2. Validate transfer command
3. Nếu command hợp lệ, tiến hành cập nhận số dư của 2 account tương ứng. Chú ý rằng trong một cluster, dữ liệu của các accounts có thể thuộc về các nodes khác nhau.

Wallet service là **stateless** nên nó dễ dàng được **scale horizontally**.

![Screenshot 2024-02-13 at 8 01 35](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c7dcb68e-4cc5-49e9-b4e1-d5337a5a46c6)

Ở hình trên chúng ta có 3 redis nodes với 3 clients là A, B, C. Các account này sẽ nằm rải rác ở cả 3 redis nodes.

Ta sẽ có 2 wallet service nodes xử lí các **account transfer command**. Khi có một transfer command (VD: chuyển $1 từ A sang B), sẽ có 2 commands phát sinh.

- Command 1: lấy $1 từ A
- Command 2: gửi $1 cho B

2 commands này sẽ tác động lên 2 redis nodes khác nhau.

Wallet service sẽ tìm ra node chứa account để cập nhật dữ liệu, nó sẽ lấy các thông tin này (ở đây gọi là **sharding information**) từ Zookeeper. Thế nhưng có những trường hợp "lưng chừng", đó là khi command thành công cập nhật account A nhưng sau đó thì wallet service bị crashed nên account B không được cập nhật, dẫn đến trình trạng `incomplete command`, để đảm bảo tình trạng này không xảy ra ta cần chắc chắn rằng **hai thao tác cập nhật đều thuộc về atomic transaction**.

### Distributed transactions

#### Database sharding

Làm cách nào để cập nhật dữ liệu trên nhiều storage node một cách "atomic", câu trả lời đó ra gộp chung các redis nodes lại thành một RDB duy nhất sau đó sẽ phân vùng (partition) RDB node này ra thành các shared như hình dưới đây.

![Screenshot 2024-02-13 at 22 37 52](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/f2f10baf-d0cb-4d2f-9121-b4b53dded085)

Việc sử dụng RDB chỉ có thể giải quyết một phần của vấn đề khi ta không thể đảm bảo rằng các xử lí cập nhật dữ liệu sẽ được thực hiện tại cùng một thời điểm.

Nếu wallet service restart sau khi cập nhật dữ liệu của account A, thì bằng cách nào chúng ta có thể đảm bảo được việc cập nhật dữ liệu của account B cũng sẽ được thực hiện ?

#### Distributed transaction: Two-phase commit

Trong một hệ thống phân tán, một transaction có thể thực hiện nhiều xử lí trên nhiều nodes khác nhau. Có 2 cách để triển khai distributed transaction:

- Low-level solution
- High-level solution

Low-level solution thường sẽ dựa trên bản thân DB, giải thuật thường được sử dụng ở đây đó là `two-phase commit (2PC)`.

![Screenshot 2024-02-19 at 8 00 21](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c9af8e5b-db41-4aa7-8eb0-47ba009b716a)

1. Coordinator (ở đây là wallet service) tiến hành đọc và ghi cùng lúc trên nhiều DB, như hình trên DB A và C đều được lock lại.
2. Khi app chuẩn bị commit transaction, coordinator sẽ yêu cầu các DB về việc "chuẩn bị" transaction.
3. Ở phase thứ 2, coordinator sẽ thu thập câu trả lời từ các db và thực hiện
   a. Nếu mọi DB trả lời `yes`, coordinator sẽ yêu cầu DB commit các transactions.
   b. Nếu có DB trả lời `no`, coordinator sẽ yêu cầu DB abort transaction đó.

Vấn đề lớn nhất với 2PC đó chính là "hiệu năng", khi lock sẽ được giữ trong một khoảng thời gian dài khi chờ message từ các nodes khác.

Một vấn đề khác nữa với 2PC đó là coordinator có thể là SPOF, được minh hoạ như hình bên dưới

![Screenshot 2024-02-19 at 8 09 36](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/5dc91341-af0d-4fdf-81a9-482391be4edf)

#### Distributed transaction: Try-Confirm/ Cancel (TC/C)

TC/C có 2 bước:

1. Ở pha đầu tiên, coordinator sẽ yêu cầu tất cả các DB chuẩn bị resources cho transaction.
2. Ở pha thứ hai, coordinator sẽ lấy về toàn bộ replies từ các DB
   a. Nếu mọi DB đều reply `yes`, coordinator sẽ yêu cầu các DB xác nhận thao tác (Try-Confirm process).
   b. Nếu có bất kì DB nào reply `no`, coordinator sẽ yêu cầu tất cả các DB cancel thao tác (Try-Cancel process).

Chú ý rằng các pha trong 2PC sẽ được bao ngoài bởi cùng 1 transaction nhưng mỗi pha trong TC/C sẽ là một transaction riêng.

**Ví dụ về TC/C**

Một ví dụ về TC/C trong thực tế về việc chuyển khoản từ account A sang account C.

Pha 1: `Try` - Account A (Balance -$1), Account C (Do nothing)
Pha 2:

- `Confirm` - Account A (Do nothing), Account C (Balance +$1)
- `Cancel` - Account A (Balance +$1), Account C (Do nothing)

**Pha 1**: trong pha này, wallet service sẽ đóng vai trò như một coordinator, nó sẽ gửi 2 transaction commands đến 2 DB

1. Với DB lưu account A, coordinator sẽ bắt đầu một local transaction giảm đi $1
2. Với DB lưu account C, coordinator sẽ gửi một NOP (no operation) command đến DB, DB sẽ không làm gì với `NOP command` này cả và luôn reply success message.

![Screenshot 2024-02-20 at 8 39 25](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/c9dee549-5a86-480f-bda5-00f8cbfa6b90)

**Pha 2 - Confirm**: nếu cả 2 databases đều reply `yes`, wallet service sẽ bắt đầu pha Confirm

Ở pha đầu tiên, account A đã bị -$1, nhưng account C vẫn chưa thay đổi, ở pha confirm, account C sẽ được +$1

![Screenshot 2024-02-20 at 23 22 08](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ca95b538-478f-49dc-9edf-5c969e58f341)

**Pha 2 - Cancel**: trong thực tế, `NOP` operation có thể fail, có thể do account C đã quá cũ nên tiền không được phép "chảy vào" cũng như "chảy ra" khỏi nó. Trong trường hợp này `distributed transaction` sẽ phải canceled.

Do sau pha đầu tiên, account A đã được cập nhật (transaction của pha 1 đã hoàn tất) nên ta không thể cancel được. Ta chỉ có thể tạo một transaction mới với nhiệm vụ "hoàn lại tiền" cho account A.

Do account C không thay đổi gì cả nên wallet service chỉ cần gửi `NOP` operation đến cho account C database.

![Screenshot 2024-02-20 at 23 27 40](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/0af1ad87-cb93-4b56-a7aa-22551e087e3d)

#### So sánh 2PC và TC/C

Trong 2PC mọi local transactions vẫn ở trạng thái lock (chưa hoàn tất) khi pha thứ 2 bắt đầu.

Trong TC/C mọi local transaction đều được hoàn tất (unlocked) khi pha thứ 2 bắt đầu.

Trong trường hợp **pha thứ 2 thành công**:

- 2PC: commit mọi local transactions
- TC/C: thực thi local transaction mới nếu cần.

Trong trường hợp **pha thứ 2 fail**:

- 2PC: cancel mọi local transactions.
- TC/C: reverse lại mọi local transactions đã kết thúc trước đó (dưới hình thức một transaction mới).

TC/C là một **high-level solution**, nó được triển khai ở business logic.

Một nhược điểm của TC/C đó là ở tầng `business logic` cũng như `application layer` ta cần các xử lí phức tạp của `distributed transaction`.

### Phase status table

Vẫn còn một câu hỏi mà chúng ta vẫn chưa trả lời được đó là "Điều gì sẽ xảy ra nếu wallet service restart giữa chừng TC/C ?", khi nó restart, mọi thao tác trước đó sẽ bị mất và hệ thống sẽ không có cách nào để recover cả.

Giải pháp ở đây đó là "lưu trữ progress của TC/C như phase status trong transactional database", phase status sẽ bao gồm các thông tin sau:

- ID, nội dung của distributed transaction.
- Status của try phase trên mỗi DB (`not sent yet`, `has been sent`, `response received`).
- Tên của phase thứ 2 (`Confirm` hoặc `Cancel`).
- Status của phase thứ 2.
- Out-of-order flag

Chúng ta nên lưu `phase-status-table` trong DB chứa account mà tiền đi ra khỏi đó.

![Screenshot 2024-02-21 at 7 34 43](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ad3cdeb6-cf1d-457b-9605-43c21c09b8db)

### Unbalanced state

![Screenshot 2024-02-23 at 17 04 19](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ff33a457-f980-4940-b824-7b91c1d6802b)

Quan sát hình ở phía trên, chúng ta sẽ thấy rằng sau phase thứ nhất thì tổng tiền ở account A và C là `$0` do account A - $1 nhưng account C chưa nhận được tiền, điều này vi phạm quy tắc cơ bản của `accounting` đó là `tổng phải giống nhau trước và sau một transaction`

Nhưng một tin tốt đó là các yếu cầu cơ bản của transaction vẫn được đảm bảo bởi TC/C, TC/C đảm bảo các yếu tố của local transaction. Do TC/C được vận hành ở application layer, bản thân application layer có thể thấy được các kết quả trung gian giữa các local transaction.

### Valid operation orders

Có 3 sự lựa chọn cho `Try phase`

1. account A: -$1, account C: NOP
2. account A: NOP, account C: +$1
3. account A: -$1, account C: +$1

Với sự lựa chọn thứ 2, giả sử try phase thành công với account C nhưng lại failed với account A, việc ta cần làm là thực thi Cancel phase, thế nhưng nếu một ai đó lại lấy đi $1 từ account C thì account C lúc này không còn lại gì cả, do đó hệ thống lúc này đã vi phạm quy tắc của distributed transaction.

Với sự lựa chọn thứ 3, khi này ta tiến hành thực hiện việc lấy $1 từ A và thêm $1 sang C một cách đồng thời. Nếu việc thêm $1 vào C thành công nhưng việc lấy $1 từ A lại thất bại thì cách xử lí ở đây là gì ?

Do đó sự lựa chọn đầu tiên là "an toàn" hơn cả.

### Out-of-order execution

Một side-effect của TC/C đó là `out-of-order execution`.

Xét ví dụ sau:
![Screenshot 2024-02-23 at 17 27 53](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/cafa0e74-6b84-47ee-8be6-36ea74745c51)

Sử dụng lại ví dụ về chuyển $1 từ account A sang account C, như hình ở trên, khi Try phase được thực thi, nhưng do sự cố nên thao tác lấy tiền từ A bị fail, Try phase kết thúc và chuyển qua Cancel phase. Ở Cancel phase, cancel operation được gửi cho cả account A và account C.

Giả sử rằng database xử lí account C gặp sự cố mạng và nhạn được cancel instruction trước try instruction nên do đó không có gì được cancel.

Để xử lí out-of-order operation, mỗi node được cho phép cancel TC/C mà không cần nhận Try instruction nào, xử lí cụ thể như sau:

- Out-of-order Cancel operation sẽ để lại một flag trong DB, chỉ ra rằng **nó đã thấy Cancel operation và chưa thấy bất kì try operation nào**.
- Try operation sẽ liên tục kiểm tra xem có `out-of-order flag` nào không, nếu có nó sẽ trả về failed.

### Distributed transaction: SAGA

#### Linear order execution

SAGA là một chuẩn để thực thi distributed transaction trong microservice architecture. Ý tưởng của Saga rất đơn giản:

1. Tất cả các thao tác đều được thực hiện một cách tuần tự, mỗi một thao tác sẽ là một transaction độc lập trên DB của nó.
2. Các thao tác được thực thi từ đầu đến cuối, hết thao tác này đến thao tác khác.
3. Khi một thao tác failed, ta cần rollback từ thao tác hiện tại cho đến thao tác đầu tiên theo thứ tự ngược lại, sử dụng phương thức "đền bù transaction - compensating transaction", giả sử ta có n thao tác, ta cần chuẩn bị tất cả **2n transaction** - n transaction cho n thao tác và n transaction còn lại cho quá trình rollback.

Mô tả bằng sơ đồ như sau:

![Screenshot 2024-02-23 at 18 23 59](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/6fb5ef51-d611-42e9-afa3-f12662d61a39)

2 trục dọc sẽ là các tình huống lỗi xảy ra. Như đã đề cập trong phần "Valid operation orders", ta luôn thực hiện thao tác trừ tiền trước thao tác cộng tiền.

Có 2 cách để tiến hành "cộng tác" giữa các microservice

1. Choreography: các distributed transaction sẽ subscribe các events của các các services khác.
2. Orchestration: một coordinator sẽ "chỉ đạo" các services khác làm việc theo một thứ tự thích hợp.

Choreography sẽ được thực thi full async nên sẽ khó kiểm soát các internal state machine khi có nhiều service.

Orchestration có khả năng xử lí phức tạo tốt, do đó đây là giải pháp thường dùng trong digital wallet system.

### So sánh TC/C và Saga

Cả TC/C và Saga đều là distributed transactions ở application-level.

Thế nhưng mỗi loại sẽ có những đặc điểm riêng như sau:

Với Saga:

- Luôn thực thi theo thứ tự tuần tự (không thể thực thi song song).
- Có thể thấy được trạng thái "lưng chừng" của transaction.
- Với Orchestration Mode sẽ có `Central coordination`.

Với TC/C:

- Có thể thực thi theo bất kì thứ tự nào.
- Có khả năng thực thi song song
- Có thể thấy được trạng thái "lưng chừng" của transaction.
- Có `Central coordination`.

Tuỳ vào yêu cầu về độ trễ - latency requirement ta có thể đưa ra sự lựa chọn phù hợp.

Với hệ thống nhỏ ta có thể sử dụng hoặc là TC/C hoặc là Saga. Nhưng nếu muốn đi theo mô hình micro-service thì nên lựa chọn Saga

Thế nhưng với những hệ thống yêu cầu cao về độ trễ thì TC/C sẽ là một sự lựa chọn tốt hơn.

Tuy nhiên ta cũng cần lưu ý rằng do TC/C và Saga được implement ở application-layer nên nó sẽ bị ảnh hưởng bởi thao tác "nhập liệu lỗi" từ phía user nên ta cần một cơ chế để có thể truy vết về nguyên nhân gốc rễ của vấn đề. `Event sourcing` sẽ giúp chúng ta có thể giải quyết bài toán nêu trên.

### Event Sourcing

#### Background
