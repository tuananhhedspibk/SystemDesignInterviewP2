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
