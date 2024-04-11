# Chương 10: Thiết kế Real-time Gaming Leaderboard

Leaderboard rất phổ biến trong giới gaming, nó cho chúng ta thấy ai đang là người đứng đầu trong các cuộc chơi. Có thể là user dành được nhiều điểm nhất hoặc hoàn thành nhiều tasks hay challenge nhất.

![Screenshot 2024-04-12 at 8 18 14](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e15915e2-26f4-493c-b30f-3f8f86dc8c1a)

## Bước 1: Hiểu vấn đề và phạm vi thiết kế

- Cơ chế tính point cho user khá đơn giản, khi user thắng một match game, họ sẽ nhận được point.
- Mọi users phải có mặt trong leaderboard.
- Mỗi tháng, khi một giải game mới kích hoạt, chúng ta sẽ bắt đầu một leaderboard mới.
- Chúng ta giả sử rằng, sẽ có 10 users top đầu được hiển thị.
- DAU khoảng 5 triệu, MAU khoảng 25 triệu.
- Có khoảng 10 game sẽ được chơi mỗi ngày trong một mùa giải.
- Khi 2 users có cùng point, họ sẽ có cùng rank (có thể đi sâu hơn nếu có thời gian).
- Leaderboard cần real-time.

### Non-functional requirements

- Real-time update
- Scalability, availability, reliability.

### Back-of-the-envelope estimation

Với 5 triệu DAU, trong khoảng 24-giờ ta sẽ có `5,000,000 DAU / 10^5 (s) = 50` users mỗi giây.

Trên thực tế việc users chơi game sẽ không được phân bổ đều mà sẽ có những khoảng thời gian peak khi nhiều users từ các múi giờ khác nhau cùng chơi game. Chúng ta có thể giả sử rằng peak load sẽ gấp 5 lần lưu lượng trung bình.

Nên do đó peak load ở đây sẽ là 250 user trên giây.
