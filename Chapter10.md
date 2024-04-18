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

| **Field** | **Description**                                    |
| --------- | -------------------------------------------------- |
| user_id   | user thắng game                                    |
| points    | Số điểm mà người chơi nhận được khi thắng một game |

Đây là internal API chỉ có thể được gọi bởi server. Client không được phép gọi API này trực tiếp.

Response sẽ như sau:

| **Name** | **Description**     |
| -------- | ------------------- |
| 200      | Cập nhật thành công |
| 400      | Cập nhật thất bại   |

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

| **Field** | **Description** |
| --------- | --------------- |
| user_id   | id của user     |

Sample response:

```json
{
  "user_id": "user_1",
  "rank": 1,
  "score": 976
}
```

### High-level architecture

Như hình dưới đây chúng ta thấy:

![Screenshot 2024-04-12 at 22 02 02](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/81c512c8-0f96-4540-8ffa-ebc5326b90bd)

Có 2 services:

- Game service: cho phép người dùng chơi game
- Leaderboard service: tạo và hiển thị leaderboard.

1. Khi người chơi thắng 1 game, client gửi request đến game service.
2. Game service đảm bảo việc người chơi chiến thắng là hợp lệ, sau đó nó sẽ gọi leaderboard service để update score.
3. Leaderboard service cập nhật score của user trong leaderboard store.
4. Người chơi sẽ gọi trực tiếp đến leaderboard service trực tiếp để lấy về leaderboard data, nó bao gồm:

- Top 10 leaderboard.
- Rank của mỗi người chơi trên leaderboard.

#### Liệu client có nên gọi trực tiếp leaderboard service hay không ?

![Screenshot 2024-04-12 at 22 12 09](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/3e2c4091-6893-44b6-bf95-ba1856d2db66)

Trong thiết kế thay thế ở hình trên, score sẽ được set ở phía client. Lựa chọn này không an toàn một chút nào vì có thể bị ảnh hưởng bởi man-in-the-middle attack, do đó score nên được set ở phía server.

Chú ý rằng với các `server authorilative game` như `online poker`, client có thể không cần phải gọi trực tiếp lên game server để set score vì game server xử lí tất cả các game logic và nó biết khi nào game kết thúc và có thể tự set score mà không cần tương tác từ phía client.

#### Liệu có cần message queue giữa game service và leaderboard service ?

Câu trả lời ở đây là tuỳ vào cách game score được sử dụng. Nếu dữ liệu được sử dụng ở các nơi khác hoặc hỗ trợ nhiều tính năng, việc đưa dữ liệu vào Kafka như hình dưới đây là hoàn toàn cần thiết.

![Screenshot 2024-04-12 at 22 18 56](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/602410f7-2b9e-48bf-8b34-fb54e6787e16)

Bằng cách này dữ liệu sẽ được sử dụng bởi nhiều services như:

- Leaderboard service.
- Analytic service.
- Push Notification service.

Điều này đặc biệt đúng khi là turn-based hoặc multi-player game.

Chú ý rằng nếu đây không phải là yêu cầu từ phía interviewer, đừng sử dụng message queue trong thiết kế.

### Data models

Một trong những component quan trọng của hệ thống đó là leaderboard store. Chúng ta sẽ nói về các phương án tiềm năng như:

- RDB
- Redis
- NoSQL

#### RDB

Hãy thử tưởng tượng trường hợp đơn giản ở đây khi việc scale không quá quan trọng và chúng ta chỉ có một vài users ?

Khi user thắng 1 game, chúng ta chỉ đơn thuần là tăng giá trị của cột score lên 1.
Với user rank, chúng ta sẽ sắp xếp theo cột score với thứ tự giảm dần.

![Screenshot 2024-04-12 at 22 32 58](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/3475c258-9382-4fef-b460-1c1046a244d7)

Trong thực tế, bảng leaderboard sẽ có các thông tin khác như `game_id` hay timestamp, ...

Để đơn giản chúng ta giả sử rằng chỉ có leaderboard data của tháng hiện thời được lưu trong bảng leaderboard.

##### Khi user thắng game

![Screenshot 2024-04-14 at 16 20 57](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/e8d4cfdf-e475-4a86-9020-e6d30906fb63)

Giả sử mọi score update đều được tăng 1, nếu user chưa có entry trong leaderboard của tháng, entry đầu tiên sẽ được insert:

```SQL
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

Câu update dữ liệu sẽ như sau:

```SQL
UPDATE leaderboard SET score = score + 1 WHERE user_id = 'mary1934';
```

##### Tìm vị trí của user trong leaderboard

![Screenshot 2024-04-14 at 16 25 36](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/22893954-ea40-4ea9-ae47-972d55321c60)

Để lấy về user rank, chúng ta có thể sắp xếp leaderboard table và rank theo score như sau:

```SQL
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

![Screenshot 2024-04-14 at 16 28 57](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/f4eaf379-2a0b-48a3-b8ef-60b1fff2bf21)

Giải pháp này chỉ phát huy tác dụng khi số lượng bản ghi là nhỏ. Khi số lượng bản ghi lớn hơn thì hiệu năng là điều mà ta cần cân nhắc.

Thế nhưng trong thực tế có những user sẽ có cùng điểm vậy nên việc xác định vị trí cho user không chỉ đơn thuần là dựa theo điểm.

SQL DB không quá mạnh khi chúng ta phải xử lí một lượng lớn các thông tin thay đổi liên tục, việc xử lí cả triệu bản ghi có thể sẽ mất khoảng 10s - đây là điều không thể chấp nhận được đối với một ứng dụng yêu cầu tính real-time cao.

Do dữ liệu liên tục thay đổi nên việc sử dụng cache sẽ không đem lại hiệu quả gì nhiều cả.

Với lưu lượng read queries cao thì RDB không phải là giải pháp phù hợp. RDB sẽ phù hợp hơn với các `batch operation`.

Một giải pháp tối ưu hơn đó là chúng ta có thể thêm index và giới hạn số lượng pages để scan mới `LIMIT` như sau:

```SQL
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

Cách tiếp cận này không có khả năng scale cao. Đầu tiên, việc tìm user rank yêu cầu phải scan toàn bộ bảng để biết được rank. Thứ hai, cách tiếp cận này không cung cấp một giải pháp trực tiếp cho việc tìm ra rank của user không nằm trên top của leaderboard.

#### Giải pháp Redis

Chúng ta cần tìm một giải pháp cho cả triệu users và cho phép chúng ta khả năng triển khai dễ dàng các leaderboard operations.

Redis có thể cung cấp cho chúng ta một giải pháp hữu ích, redis là một `in-memory data store` hỗ trợ key-value pairs. Do là in-memory nên nó hỗ trợ fast-read và write.

Redis có một kiểu dữ liệu gọi là `sorted sets` - rất lí tưởng để giải quyết vấn đề của chúng ta.

##### Sorted sets là gì ?

Là một kiểu dữ liệu tương tự như set. Mỗi phần tử của sorted set sẽ tương ứng với score. Các phần tử của set sẽ là unique nhưng score thì có thể lặp lại. Score được sử dụng để rank sorted set sẽ có thứ tự tăng dần.

Sorted set được triển khai bằng hai cấu trúc dữ liệu:

- Hash table.
- Skip list.

Hash table sẽ map user với score và Skip list map score tới user.

Trong sorted set, user được sắp xếp theo score.

![Screenshot 2024-04-14 at 22 20 41](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/4fba6022-9be5-4b83-b051-80bfa37bb5fd)

Skip list là một cấu trúc list cho phép tìm kiếm nhanh. Nó bao gồm:

- Sorted linked list.
- Multi-level indexes.

![Screenshot 2024-04-14 at 22 28 59](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/d9ad30f5-a8fd-4f6c-8d32-0aa8f6de661c)

Như ở hình ví dụ trên đây, độ phức tạp thời gian của phép insert, remove và tìm kiếm là `O(n)`.

Có một cách để làm cho các thao tác này nhanh hơn đó là đi đến vị trí middle nhanh nhất có thể - giống như binary search làm. Để thực hiện điều này chúng ta thêm level 1 index bỏ qua 1 số nodes, level 2 index bỏ qua một số nodes ở level 1. Các level tiếp theo sẽ bỏ đi một vài nodes của level trước đó. Chúng ta sẽ dừng việc thêm các levels mới cho đến khi khoảng cách giữa các node là `n/2 - 1`, với n là tổng số các nodes.

Như chúng ta cũng có thể thấy ở hình trên thì việc tìm đến node 45 nhanh hơn rất nhiều khi chúng ta có multi-level indexes.

Khi tập dữ liệu còn nhỏ thì việc cải thiện tốc độ thông qua việc skip node có thể không để lại ấn tượng gì sâu sắc nhưng khi ta có 5 levels indexes thì việc cải thiện là hoàn toàn rõ rệt, như ở hình dưới ta thấy rằng với base linked list, ta cần đi qua 62 nodes để đến được node muốn tìm kiếm, còn với skip list ta chỉ cần đi qua 11 nodes.

![Screenshot 2024-04-15 at 8 14 51](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/1f1ffd51-62cd-46d7-84d7-840c5b7d8ec5)

Sorted set đem lại hiệu năng tốt hơn RDB vì bản thân các element sẽ được tự động sắp xếp đúng thứ tự khi chúng được insert, update và ngoài ra thì độ phức tạp thời gian của các thao tác này với sorted set cũng chỉ là hàm loga `O(log(n))`.

Ngược lại, khi muốn lấy ra rank của user trong RDB chúng ta cần thực hiện nested query như sau:

```SQL
SELECT *, (SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id}
```

##### Triển khai với Redis sorted sets

Với leaderboard của mình chúng ta sẽ sử dụng những thao tác sau đây của Redis:

- `ZADD`: insert user vào set nếu user chưa tồn tại. Ngược lại, sẽ update score cho user, thao tác này tốn `O(log(n))`.
- `ZINCRBY`: tăng user score, nếu user chưa tồn tại, nó sẽ giả sử score bắt đầu từ 0, thao tác này tốn `O(log(n))`.
- `ZRANGE / ZREVRANGE`: lấy về range users được sắp xếp bởi score. Chúng ta có thể chỉ định order (range vs revrange), số lượng entries, vị trí bắt đầu. Thao tác này tốn `O(log(n) + m)` với m là số lượng entries sẽ lấy về, n số lượng entries có trong set.
- `ZRANK / ZREVRANK`: lấy về vị trí của bất kì user nào theo thứ tự tăng/ giảm dần trong thời gian logarithmic.

##### Làm việc với sorted sets

1. User ghi điểm

![Screenshot 2024-04-17 at 8 38 16](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/267ea716-7f77-4bd1-8921-9ee7caf49329)

Mỗi tháng, chúng ta sẽ tạo một leaderboard sorted set và phần của tháng trước sẽ được đưa vào historical data storage. Khi user thắng một trận, họ sẽ được nhận thêm 1 điểm, do đó chúng ta gọi `ZINCRBY` - cộng 1 point cho user hoặc thêm user vào leaderboard nếu user chưa tồn tại. Cú pháp cho `ZINCRBY` sẽ là:

```sh
ZINCRBY <key> <increment> <user>
```

2. User lấy về top 10 global leaderboard

![Screenshot 2024-04-17 at 22 43 11](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/93e47e68-d96c-4985-8652-bdf3fea81cfc)

Chúng ta sử dụng `ZREVRANGE` để lấy về members theo thứ tự giảm dần vì chúng ta muốn điểm cao nhất, truyền `WITHSCORES` attribute để đảm bảo rằng trả về tổng số điểm của mỗi user.

```sh
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

Câu lệnh trên sẽ trả về top 10 players với số điểm cao nhất của `leaderboard_feb_2021` board. Kết quả trả về sẽ như sau:

```txt
[(user2, score2), (user1, score1), ...]
```

3. User lấy về vị trí của họ

![Screenshot 2024-04-17 at 22 47 42](https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/5e8f9000-b04b-4994-8b92-0d09dc73e47f)

Chúng ta gọi `ZREVRANK` để lấy về rank của user trên leaderboard. Chúng ta gọi `rev version` của câu lệnh vì chúng ta muốn sắp xếp kết quả từ cao xuống thấp.

```sh
ZREVRANK leaderboard_feb_2021 'mary1934'
```

4. Lấy về vị trí tương đối của user trên leaderboard

<img width="569" alt="Screenshot 2024-04-18 at 8 19 03" src="https://github.com/tuananhhedspibk/tuananhhedspibk.github.io/assets/15076665/95441868-8ea9-42e5-9255-a8b1d1130996">

Đây không phải là một yêu cầu bắt buộc nhưng chúng ta có thể dễ dàng lấy về vị trí tương đối cho user bằng việc tận dụng `ZREVRANGE` với số lượng kết quả trên và dưới user. Ví dụ, nếu user `Mallow007` có rank là 361, chúng ta muốn lấy về 4 người chơi trên và dưới user này, ta có thể chạy câu lệnh như sau:

```sh
ZREVRANGE leaderboard_feb_2021 357 365
```

##### Storage requirement

Tôí thiểu, chúng ta cần lưu user id và score. Trường hợp tệ nhất là khi tất cả 25 triệu MAU đều thắng ít nhất 1 game và tất cả user đều có entries trong leaderboard của tháng. Giả sử id là string có 24 kí tự, score là số nguyên 16-bit, chúng ta cần 26 bytes lưu trữ cho mỗi một bản ghi trong leaderboard.

Trong trường hợp tệ nhất, chúng ta cần `26 bytes x 25 triệu = 650 triệu bytes ~ 650MB` cho leaderboard lưu trữ trong Redis cache. Kể cả khi chúng ta cần gấp đôi lượng bọ nhớ để đảm bảo skip list thì một Redis server hiện đại vẫn có thể đáp ứng đầy đủ được.

Một yếu tố khác cần cân nhắc đó là CPI và I/O usage. Peak QPS ở đây là `2500 updates/s` - bản thân một Redis server đơn có thể đảm bảo được thông số này với một hiệu năng ổn.

Redis đảm bảo lưu trữ "vĩnh viễn", nhưng việc restart Redis instance khá chậm. Thông thường redis được thiết lập với một read replica và khi main instance failed, read replica sẽ được đưa thành main và một read replica mới sẽ được thêm vào.

Ngoài ra chúng ta cần 2 bảng (user và point) trong RDB giống như MySQL. Bảng user sẽ bao gồm `user ID` và `user display name` (trong thực tế có thể có nhiều thuộc tính hơn), bảng point sẽ bao gồm `user ID`, `score`, `timestamp khi user tắng game`.
