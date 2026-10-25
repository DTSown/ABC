# 03. MOVE và xử lý va chạm

## 1. Luồng code cần thuộc

```text
GamePanel.createMoveButton()
→ MainFrame tạo Message(MOVE, direction)
→ ClientConnection gửi dòng TCP
→ ClientHandler nhánh MOVE
→ ServerManager.move()
→ GameRoom.handleMove()
→ ServerManager gửi GAME_STATE
```

## 2. Client nhận thao tác

`GamePanel.createMoveButton()` tạo một `Action` cho mỗi hướng.

| Thao tác | Direction gửi đi |
|---|---|
| W hoặc ↑ | `UP` |
| S hoặc ↓ | `DOWN` |
| A hoặc ← | `LEFT` |
| D hoặc → | `RIGHT` |

Nút và phím dùng chung một Action nên logic không bị lặp.

Điểm quan trọng:

```java
moveSender.accept(direction);
```

Client chỉ gửi direction. Không có dòng nào tại đây tự thay đổi `Position`.

## 3. Message trên mạng

Ví dụ:

```text
MOVE|UP
```

Trong `ClientHandler`:

```java
case Protocol.MOVE:
    serverManager.move(this, message.getData());
    break;
```

`this` là handler của socket gửi request. Nhờ đó Server biết người gửi thật từ phiên đăng nhập, thay vì cho Client tự gửi username.

## 4. Xác thực trong `ServerManager.move()`

### Kiểm tra phiên đăng nhập

```java
clientsByUsername.get(sender.getClientUsername()) == sender
```

Không chỉ so username; Server còn so đúng object handler. Một kết nối cũ không thể điều khiển phiên mới có cùng username.

### Kiểm tra trạng thái

Player phải là `PLAYING`. Người đang Lobby hoặc đã kết thúc trận không được MOVE.

### Tìm đúng Room

Server duyệt `gameRooms` và so identity:

```java
room.getPlayer1() == player || room.getPlayer2() == player
```

Nếu không tìm thấy, trả lỗi “Bạn không thuộc phòng thi đấu nào”.

## 5. Khóa Room

Sau khi tìm đúng Room:

```java
synchronized (room) {
    // handleMove
    // collectTreasure
    // respawn
    // broadcast
}
```

Khóa bao trọn toàn bộ giao dịch gameplay. Không có Client khác chen vào giữa bước đổi vị trí và bước kiểm tra rương.

## 6. `GameRoom.handleMove()`

### Giá trị trả về

| Giá trị | Ý nghĩa |
|---|---|
| `null` | Bước đi hợp lệ, Position đã được cập nhật |
| Chuỗi lỗi | Bước đi bị từ chối, state giữ nguyên |
| `MOVE_STUNNED` | Mở rộng bẫy hiện tại; MOVE bị bỏ qua khi choáng |

### Xác định người di chuyển

Server so username với hai Player trong `GameState`. Nếu không trùng ai, từ chối.

### Tính ô đích

Giả sử vị trí hiện tại là `(row, col)`:

```text
UP    → (row-1, col)
DOWN  → (row+1, col)
LEFT  → (row, col-1)
RIGHT → (row, col+1)
```

### Ba nhóm va chạm

1. Ngoài biên:

```text
row < 0 hoặc row >= 7 hoặc col < 0 hoặc col >= 7
```

2. Vật cản:

```text
map[row][col] == 1
```

3. Đối thủ:

```text
opponent.position == destination
```

Chỉ khi vượt qua toàn bộ điều kiện, Server mới gọi:

```java
moving.setPosition(new Position(row, col));
```

## 7. Ví dụ cụ thể

### Đi hợp lệ

Alex ở `(3,3)`, gửi `RIGHT`, ô `(3,4)` là đường và không có Nina.

Kết quả: Alex chuyển tới `(3,4)` và Server tiếp tục kiểm tra rương.

### Đâm tường

Alex ở `(3,3)`, gửi `UP`, ô `(2,3)` có giá trị `1`.

Kết quả: `handleMove()` trả “Di chuyển không hợp lệ”; Position vẫn là `(3,3)`.

### Hai người cùng đi vào một ô

Alex và Nina gửi request từ hai thread khác nhau. Khóa Room cho phép một request chạy trước.

- Request đầu chiếm ô đích.
- Request sau nhìn thấy đối thủ đã ở ô đó và bị từ chối.

Hai người không thể chồng lên nhau.

## 8. Server gửi state mới

Sau MOVE hợp lệ, dù có nhặt rương hay không, Server encode `room.getGameState()` và gửi cho hai Client.

Do thay đổi state và hai lần gửi nằm dưới cùng khóa Room:

- hai Client quan sát các snapshot theo cùng thứ tự;
- không có state giữa chừng;
- request tiếp theo chỉ xử lý sau khi request trước hoàn tất phần gameplay.

## 9. Các request giả bị chặn

### Client gửi tọa độ thay direction

Ví dụ `MOVE|3,4` không thuộc bốn hướng, `handleMove()` trả lỗi.

### Client tự gửi `GAME_STATE`

`ClientHandler` xác định đây là message chỉ Server được gửi và trả `ERROR`.

### Client không đăng nhập gửi MOVE

`ServerManager.move()` không tìm thấy phiên hợp lệ và từ chối.

### Client ở Room khác gửi MOVE

Server tìm Room theo object Player của handler, nên không thể chỉ định Room của người khác trong payload.

## 10. Câu trả lời mẫu

### Vì sao không cập nhật vị trí ngay trên Client?

> Nếu Client tự cập nhật, hai máy có thể lệch nhau và người chơi có thể sửa Client để đi xuyên tường. Client chỉ gửi direction; Server kiểm tra và gửi state xác nhận về cả hai.

### Hai người MOVE đồng thời thì sao?

> Mỗi Client có thể gửi từ thread riêng, nhưng phần xử lý state được đặt dưới khóa Room. Các request chạy tuần tự; snapshot và thứ tự gửi tới hai Client luôn nhất quán.

