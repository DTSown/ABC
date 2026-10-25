# 05. Đồng bộ GAME_STATE

## 1. Mục tiêu đồng bộ

Sau mỗi thay đổi hợp lệ, cả hai Client phải thấy giống nhau về:

- bản đồ và vật cản;
- vị trí Player 1 và Player 2;
- điểm của từng người;
- số rương đã nhặt;
- vị trí và loại rương hiện tại;
- thời gian và trạng thái trận.

Giải pháp của dự án là gửi **full snapshot** thay vì để Client tự suy diễn thay đổi.

## 2. Cấu trúc object `GameState`

| Field | Ý nghĩa |
|---|---|
| `roomId` | Snapshot thuộc Room nào |
| `map` | Mảng 7×7, `0` đường và `1` vật cản |
| `player1` | Username, Position, score, collected |
| `player2` | Username, Position, score, collected |
| `treasures` | Các rương đang tồn tại |
| `remainingTime` | Thời gian Server còn lại |
| `status` | `WAITING`, `RUNNING`, `FINISHED` |

## 3. Wire format

Message hoàn chỉnh:

```text
GAME_STATE|roomId;map;player1;player2;treasures;remainingTime;status
```

Payload có đúng bảy trường ngăn bởi dấu `;`.

### Map

49 chữ số `0` hoặc `1`, theo thứ tự từng hàng.

```text
map[0][0], map[0][1], ..., map[6][6]
```

### Player

```text
encodedUsername,row,col,matchScore,collectedTreasure
```

### Treasure

```text
id,type,value,row,col
```

Nhiều rương cách nhau bằng `/`.

## 4. Server encode như thế nào?

`GameStateCodec.encode()`:

1. Duyệt `map` theo hàng và nối thành 49 ký tự.
2. Encode Player 1.
3. Encode Player 2.
4. Encode từng rương rồi nối bằng `/`.
5. Ghép bảy trường bằng `;`.

Username được URL-encode để tiếng Việt hoặc ký tự phân cách không phá cấu trúc message.

## 5. Vì sao encode một lần?

Trong `ServerManager.move()`:

```java
Message state = new Message(
    Protocol.GAME_STATE,
    GameStateCodec.encode(room.getGameState())
);
sendToRoom(room, state);
```

Server tạo một payload rồi gửi cùng payload đó cho hai Client. Không encode riêng sau mỗi lần gửi, nên hai bên không thể nhận hai snapshot khác nhau do state đổi giữa chừng.

## 6. Client decode và validation

`GameStateCodec.decode()` kiểm tra theo thứ tự:

1. Payload không null.
2. Có đúng bảy trường.
3. Room ID và status không rỗng.
4. Map đúng 49 ký tự `0/1`.
5. Hai username khác nhau.
6. Điểm và số rương không âm.
7. Tất cả Position trong biên `0..6`.
8. Position nằm trên đường, không phải vật cản.
9. Hai Player và các rương không trùng ô.
10. Treasure ID không trùng.
11. Treasure value đúng với type.

Chỉ khi toàn bộ hợp lệ, Client mới nhận một `GameState` hoàn chỉnh.

## 7. Client cập nhật giao diện

### `MainFrame`

Nhánh `GAME_STATE` gọi:

```java
GameState state = GameStateCodec.decode(data);
gamePanel.setGameState(state);
```

State của Room đã có kết quả bị bỏ qua để message đến muộn không kéo Client từ Result trở lại Game.

### `GamePanel.setGameState()`

1. Nhận state mới.
2. Nếu Room ID đổi, xóa hiệu ứng tạm của trận cũ.
3. Gán `currentState`.
4. Cập nhật hai HUD.
5. Cập nhật tên Room và đồng hồ.
6. Gọi `board.setSnapshot(state, currentUsername)`.

### `BoardPanel`

`setSnapshot()` chỉ lưu state và gọi `repaint()`.

`paintComponent()` đọc trực tiếp snapshot để vẽ map, rương và hai người.

## 8. Event khác snapshot thế nào?

### Event

```text
TREASURE_COLLECTED|Alex;SILVER;20;50
```

Dùng để biết “chuyện gì vừa xảy ra” và hiện thông báo.

### Snapshot

```text
GAME_STATE|...
```

Dùng để biết “trạng thái đúng hiện tại là gì”.

Client không lấy `totalScore` từ event để sửa HUD. Điều này tránh lỗi nếu event bị lặp hoặc UI xử lý chậm.

## 9. Tại sao gửi full snapshot?

Ưu điểm trong đồ án:

- logic Client đơn giản;
- hai bên dễ đồng bộ;
- state nhỏ vì map chỉ 7×7;
- Client không cần tự áp dụng delta;
- dễ kiểm tra tính hợp lệ.

Nhược điểm:

- tốn băng thông hơn delta khi state lớn;
- encode/decode toàn bộ sau mỗi MOVE.

Với game hai người và map nhỏ, trade-off này hợp lý.

## 10. Câu trả lời mẫu

### Làm sao chứng minh hai Client cùng vị trí?

> Server cập nhật một GameState dưới khóa Room, encode đúng một payload rồi sendToRoom cho hai thành viên. Cả hai Client dùng cùng codec để decode và render nên nhận cùng vị trí, điểm và rương.

### `TREASURE_COLLECTED` có cập nhật điểm không?

> Không. Nó chỉ tạo thông báo. Điểm HUD được cập nhật từ Player nằm trong GAME_STATE tiếp theo.

