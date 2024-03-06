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

Trong thực tế, các digital wallet provider đều bị tiến hành kiểm toán, các kiểm toán viên sẽ thường xuyên hỏi các câu hỏi như:

1. Liệu chúng ta có thể kiểm tra được số dư tài khoản ở bất kì thời điểm nào ?
2. Làm cách nào chúng ta có thể biết được lịch sử và số dư tài khoản hiện thời là chính xác ?
3. Làm cách nào chúng ta có thể chứng minh được system logic là đúng kể cả sau khi code đã thay đổi ?

#### Các khái niệm

Có 4 khái niệm quan trọng trong event-sourcing

1. Command
2. Event
3. State
4. State Machine

#### Command

Là một inteded action từ bên ngoài, ví dụ: khi ta muốn chuyển $1 từ account A sang C, thì **request chuyển tiền** chính là một command.

Trong event sourcing, mọi thứ đều có tính thứ tự, command sẽ được đưa vào một FIFO queue

#### Event

Command có thể invalid hoặc không được hoàn tất. Ví dụ, transfer command có thể failed và account balance sẽ có giá trị âm sau khi transfer.

Command phải valid trước khi ta làm bất kì điều gì với nó. Khi command pass validation thì nó phải được thực thi và phải kết thúc
, kết quả của việc kết thúc một command được gọi là event.

Có 2 sự khác biệt lớn giữa command và event:

1. Event phải được thực thi vì chúng biểu thị một "validated fact", trong thực tế chúng ta thường dùng "thì quá khứ" cho event. Ví dụ nếu command là "transfer $1 từ A sang C" thì event tương ứng sẽ là "transferred $1 từ A sang C".

2. Command có thể bao hàm các yếu tố ngẫu nhiên hoặc I/O nhưng event **phải mang tính quyết định**, events sẽ biểu thị các **historical facts**.

Có hai yếu tố quan trọng của qúa trình generate event.

1. Một command có thể gen ra số lượng events tuỳ ý.
2. Event generation có thể bao gồm I/O hoặc số ngẫu nhiên nên một command sẽ gen ra các events khác nhau.

Thứ tự các events phải theo đúng như thứ tự của commands, events được lưu trong FIFO queue.

#### State

State sẽ thay đổi khi apply event. Trong wallet system, state là balances của tất cả các client accounts (có thể được biểu thị bới map data structure - key là account ID hoặc account Name, còn value chính là account balance). Key-value store thường được sử dụng để lưu map.

#### State machine

State machine sẽ điều phối event sourcing process, nó bao gồm 2 tính năng chính:

1. Validate commands và generate events.
2. Apply event để cập nhật state.

Event sourcing yêu cầu behavior của state machine cần phải "nhất quán". Do đó state machine sẽ không nhận bất kì randomness nào từ I/O. Khi apply event để cập nhật state, nó nên luôn luôn tạo ra kết quả giống nhau.

State machine có 2 nhiệm vụ chính như trên nên trong hình minh hoạ bên dưới ta sẽ thấy có 2 State machine bên trong nó.

![Screenshot 2024-03-03 at 21 47 34](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/bdddd85b-dbbd-42c8-8d62-40a46972e03b)

Nếu ta thêm time-dimension, ta sẽ được hình như bên dưới (hệ thống sẽ nhận và xử lí từng command một).

![Screenshot 2024-03-03 at 21 53 29](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/2d2d6d9e-0c8c-4be5-b486-06bab737c9f3)

#### Wallet service example

Với wallet service, command sẽ là balance transfer requests. Các commands sẽ được đưa vào FIFO queue (một lựa chọn phổ biến cho command queue là Kafka). Hình minh hoạ dưới đây

![Screenshot 2024-03-03 at 22 00 38](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/3ddddce1-876d-477b-9adb-30544e1eb4b9)

Giả sử state (account balance) được lưu trong RDB, state machine sẽ kiểm tra các commands (one by one) theo thứ tự FIFO, với mỗi command, nó sẽ kiểm tra xem balance của account có thoả mãn điều kiện hay không, nếu có nó sẽ gen ra các events tương ứng, ví dụ với command "A -$1-> C", state machine sẽ gen ra 2 events là "A: -$1" và "C: +$1"

Hình dưới đây sẽ mô tả quá trình state machine làm việc trong 5 bước:

![Screenshot 2024-03-03 at 22 47 44](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/44db28eb-7fe2-452d-897f-375d74b0df73)

1. Đọc command từ command queue.
2. Đọc balance state từ DB.
3. Validate command để từ đó gen ra các events.
4. Đọc các events.
5. Apply events bằng cách update balance trong DB.

#### Reproducibility (Khả năng tái tạo)

Một trong số những ưu điểm của event sourcing so với các kiến trúc khác đó là khả năng tái tạo.

Trong distributed transaction solution, ta sẽ lưu updated value của balance thế nhưng với event sourcing ta sẽ lưu lại "lịch sử" các sự thay đổi này (lịch sử này sẽ KHÔNG BỊ THAY ĐỔI).

Từ "lịch sử" này, ta có thể tái tạo lại balance state.

Hình dưới đây sẽ mô tả cách sử dụng event để tái tạo lại state tại một thời điểm nhất định.

![Screenshot 2024-03-03 at 22 56 18](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/5f195c4a-da58-4923-b4b5-10bc5f16ef4e)

Khả năng tái tạo này giúp chúng ta có thể "đối phó" lại với những câu hỏi dưới đây của kiểm toán:

1. Liệu chúng ta có thể biết được account balance tại bất kì thời điểm nào ?

2. Làm cách nào để có thể biết được lịch sử và balance của account hiện tại có đúng hay không ?

3. Làm cách nào chúng ta có thể chứng minh được system logic vẫn đúng ngay cả khi code thay đổi ?

Với câu hỏi đầu tiên chúng ta chỉ cần "kết tập - aggregate" các events lại cho tới một thời điểm nhất định.

Với câu hỏi thứ hai chúng ta có thể tính lại account balance bằng các event từ thời điểm ban đầu cho đến mới nhất.

Với câu hỏi thứ ba chúng ta có thể chạy các versions khác nhau của code với các events và check lại các kết quả là khác nhau.

Do đáp ứng được các yêu cầu về sự "minh bạch" nên event sourcing thường được dùng trong các wallet service.

#### CQRS

Chúng ta đã thiết kế được một wallet service chuyển tiền giữa các account nhưng client không hề biết gì về state (balance information), ta cần có một cách để publish thông tin về state tới client.

Thông thường ta có thể tạo một read-only copy cho DB và publish state. Thế nhưng event sourcing sẽ publish events mà thôi. Client có thể re-build và customize tuỳ ý. Thiết kế này được gọi là CQRS.CQRS

Trong CQRS chỉ có **một state machine** đảm nhận nhiệm vụ write, trong khi đó sẽ có **nhiều state machines** - với nhiệm vụ xây dựng views cho states, các views này sẽ được sử dụng cho mục đích queries.

Các read-only state machines có thể đem lại nhiều hình thức "biểu thị" cho event queue, ví dụ:

- State machine có thể lưu balance vào DB để client truy xuất.
- State machine có thể build lại state tại một khoảng thời gian nhất định để giải quyết các vấn đề như double-charges.

![Screenshot 2024-03-04 at 8 05 38](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/3efe72ff-cea3-403b-9a7a-0eb0f468e181)

## Bước 3: Deep Dive Design

Trong phần này chúng ta sẽ chú trọng về:

- High performance
- Reliability
- Scalability

### High performance event sourcing

Trong các ví dụ trước chúng ta sử dụng:

- Kafa để lưu command và event.
- Database để lưu state.

#### File-based command và event list

Thay vì sử dụng Kafka, ta sẽ append command và event list vào file trên ổ đĩa (các thao tác này nhanh do đã được OS tối ưu hoá) cũng như không mất thời gian cho việc truyền dữ liệu qua mạng đến remote Kafka.

Một biện pháp tối ưu thứ hai đó là cache các commands và events trong bộ nhớ. Như đã giải thích lúc trước, chúng ta sẽ xử lí commands và events ngay lập tức sau khi chúng được tạo ra, do đó việc cache trong bộ nhớ sẽ giúp giảm thời gian lấy command và event ra từ local disk.

Có một vài cách khác để triển khai như `mmap`. `Mmap` có thể ghi lên disk và memory ở cùng một thời điểm. Nó sẽ map các file từ disk sang memory dưới dạng array, các thao tác append-only đều được triển khai rất nhanh.

![Screenshot 2024-03-05 at 8 16 01](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/215cfe2c-0b77-4fdc-854d-643ed9ac3264)

#### File-based state

Khi lưu state trong DB, trong môi trường production, DB sẽ được đặt ở một server riêng biệt và phải truy cập thông qua network. Và cũng tương tự như commands và state, chúng ta cũng cần một cơ chế để tối ưu hoá việc lưu trữ các state information trong local disk.

Cụ thể hơn nữa, chúng ta có thể sử dụng SQLite SQLite - file-based RDB hoặc RockDB (local file-based key-value store).

Ta sẽ sử dụng RockDB vì nó sử dụng log-structured merge-tree (LSM) - đã được tối ưu cho thao tác ghi, để cải thiện việc đọc - read, ta có thể sử dụng cached

![Screenshot 2024-03-06 at 23 08 48](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/ad8dda56-eabc-4ae4-8356-386e62bd422d)

#### Snapshot

Đây là một giải pháp giúp chúng ta tối ưu hoá tính `repoducibility` thay vì bắt state-machine phải xử lí events từ đầu.

Với Snapshot, chúng ta sẽ lưu state vào một file. Mỗi một snapshot sẽ là một immutable view của historical state, khi snapshot được lưu, state machine sẽ không phải restart từ ban đầu nữa. State machine có thể đọc dữ liệu từ snapshot, bắt đầu xử lí từ đây.

Với finance team, họ thường yêu cầu việc lấy snapshot bắt đầu từ `00:00` với mục đích verify mọi transactions đã diễn ra trong một ngày.

Khi giới thiệu về CQRS cho event-sourcing, giải pháp ở đây đó là thiết lập read-only state machine để đọc từ thời điểm mà ta muốn bắt đầu tracking, với snapshots, state machine chỉ cần load snapshot bao gồm dữ liệu là đủ.

Snapshot thường là một binary file lớn, giải pháp cho nó sẽ là lưu nó trong object storage (ví dụ như HDFS).

Hình dưới đây minh hoạ cho `file-based event sourcing architecture`. Khi mọi thứ là file-based, hệ thống có thể tận dụng tối đa I/O throughput của hardware.

![Screenshot 2024-03-06 at 23 23 06](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/5923604d-f504-418c-a1d6-e08dc48c5d42)

Việc lưu state, command, event ở local disk sẽ làm cho server hiện thời là stateful và xảy ra SPOF, vậy làm cách nào để cải thiện tính tin cậy của hệ thống ?

### Tính tin cậy, hiệu năng cao của event-sourcing

#### Phân tích tính tin cậy

Hoạt động của mỗi một node trong hệ thống phân tán đều chỉ xoay quanh 2 khái niệm:

1. Dữ liệu
2. Computation (tính toán)

Ta có thể thực thi việc tính toán ở bất kì node nào nên điều mà ta cần lo lắng ở đây chỉ là tính tin cậy của dữ liệu vì khi dữ liệu mất, chúng ta sẽ mất tất cả. Nên do đó:

> Tính tin cậy của hệ thống chính là tính tin cậy của dữ liệu

Có 4 loại dữ liệu trong hệ thống của chúng ta:

1. File-based command
2. File-based event
3. File-based state
4. State snapshot

State và snapshot luôn luôn có thể gen lại nhờ việc "kích hoạt lại" event list, do đó để đảm bảo tính tin cậy của state và snapshot ta chỉ cần đảm bảo event list có tính tin cậy cao.

Command sẽ gen ra event thế nhưng trong quá trình gen ra event sẽ có những yếu tố ngẫu nhiên như:

- Random numbers
- External I/O
- ...

Event biểu thị lịch sử thay đổi state, event là `immutable` và có thể được sử dụng để rebuild state, do đó điều quan trọng ở đây đó là **Tính tin cậy cao của event**.

#### Tính cố kết
