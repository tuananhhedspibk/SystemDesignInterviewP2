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
