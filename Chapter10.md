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

QPS cho user ghi điểm: nếu user chơi 10 game trung bình mỗi ngày thì QPS cho việc ghi điểm sẽ là `50 x 10 = 500`. Peak QPS sẽ gấp 5 lần QPS trung bình: `500 x 5 = 2,500`.

## Bước 2: High-level design

### API design

Ở high-level chúng ta cần 3 APIs như sau:

#### POST /v1/scores

Cập nhật vị trí của user trên leader-board khi user thắng một game. Request params sẽ như sau:

| **Field**  | **Description** | 
|---|---|
| user_id |  user thắng game  |
| points | Số điểm mà người chơi nhận được khi thắng một game |

Đây là internal API chỉ có thể được gọi bởi server. Client không được phép gọi API này trực tiếp.

Response sẽ như sau:

| **Name**  | **Description** | 
|---|---|
| 200 |  Cập nhật thành công  |
| 400 | Cập nhật thất bại |

#### GET /v1/scores

Lấy về top 10 users trên leader-board

Ví dụ về response:

```JSON
{
  "data": [
    {
      "user_id": "user_1",
      "user_name": "alice",
      "rank": 1,
      "score": 976 
    },
    {
      "user_id": "user_2",
      "user_name": "bob",
      "rank": 2,
      "score": 965 
    },
  ],
  // ...
  "total": 10
}
```

#### GET /v1/scores/{:user_id}

Lấy về rank của một user cụ thể.

| **Field**  | **Description** | 
|---|---|
| user_id |  id của user  |

Sample response:

```json
{
  "user_id": "user_1",
  "rank": 1,
  "score": 976 
}
```

### High-level architecture
