# 09. Phao code và bài nói phần Sơn

## 1. Bài nói 4–5 phút

> Phần em phụ trách là gameplay trong trận, gồm sinh bản đồ, di chuyển, va chạm, nhặt rương, cộng điểm và đồng bộ trạng thái cho hai Client.
>
> Khi Room bắt đầu, Server gọi `MapGenerator.generate()` để sinh map 7×7. Server chọn bốn cặp ô đối xứng 180 độ làm tám vật cản. Sau đó chạy BFS từ tâm và chỉ chấp nhận map nếu thăm được đủ 41 ô đường, nhờ vậy không có vùng trống bị cô lập.
>
> Hai Player có spawn đối xứng. Năm rương ban đầu gồm hai rương thường 10 điểm, hai rương bạc 20 điểm và một rương vàng 30 điểm ở tâm. Toàn bộ state này được sinh ở Server, không phải ở Client.
>
> Khi người dùng bấm WASD, phím mũi tên hoặc nút điều hướng, `GamePanel` chỉ gửi direction trong message `MOVE`. `ClientHandler` nhận message và chuyển đúng phiên gửi sang `ServerManager.move()`.
>
> Server kiểm tra người gửi đã login, đang `PLAYING` và thuộc đúng Room. Dưới khóa Room, `GameRoom.handleMove()` tính ô đích rồi chặn ba trường hợp: ngoài biên, vật cản và vị trí đối thủ. Chỉ khi hợp lệ Server mới đổi Position.
>
> Sau đó Server gọi `collectTreasure()`. Nếu Player đứng trên rương thưởng, Server tự tính điểm theo type, tăng số rương, xóa rương cũ và sinh rương mới tại một ô đường không trùng Player, vật cản hoặc rương khác.
>
> Cuối transaction, Server encode một `GAME_STATE` và gửi cùng payload cho cả hai Client. Client chỉ decode, cập nhật HUD và vẽ Board. `TREASURE_COLLECTED` chỉ tạo thông báo; điểm chính thức vẫn lấy từ `GAME_STATE`.
>
> Việc di chuyển, nhặt rương, respawn và broadcast nằm trong cùng khóa Room. Vì vậy nếu hai Client thao tác đồng thời hoặc tranh cùng một rương, request được xử lý tuần tự và một rương chỉ được tính một lần. Đây là cách dự án thực hiện mô hình Server authoritative.

## 2. Bản nói 60 giây

> Client chỉ gửi hướng MOVE, không gửi tọa độ hay điểm. Server tìm đúng Player và Room, khóa Room rồi kiểm tra biên, vật cản và vị trí đối thủ. Sau bước đi hợp lệ, Server kiểm tra rương, tự cộng 10/20/30 điểm, sinh rương mới và encode một GAME_STATE gửi cho cả hai Client. Map cũng được sinh ở Server theo đối xứng 180 độ và phải vượt BFS liên thông. Vì vậy hai Client luôn hiển thị cùng trạng thái và không thể tự sửa điểm hoặc đi xuyên tường.

## 3. Phao mở code

### Thầy hỏi “Client gửi MOVE ở đâu?”

Mở:

```text
src/client/GamePanel.java
Ctrl+F: createMoveButton
```

Chỉ dòng:

```java
moveSender.accept(direction);
```

Nói: Client chỉ gửi hướng, không tự sửa Position.

### Thầy hỏi “Server nhận MOVE ở đâu?”

Mở:

```text
src/server/ClientHandler.java
Ctrl+F: case Protocol.MOVE
```

Nói: Handler truyền chính phiên socket và direction sang `ServerManager.move()`.

### Thầy hỏi “Server điều phối MOVE ở đâu?”

Mở:

```text
src/server/ServerManager.java
Ctrl+F: void move
```

Nói: kiểm tra login/PLAYING/Room, khóa Room, di chuyển, nhặt rương và broadcast.

### Thầy hỏi “Chặn đi xuyên tường ở đâu?”

Mở:

```text
src/server/GameRoom.java
Ctrl+F: handleMove
```

Chỉ điều kiện:

```java
gameState.getMap()[row][col] == 1
```

### Thầy hỏi “Cộng điểm ở đâu?”

Mở:

```text
src/server/GameRoom.java
Ctrl+F: setMatchScore
```

Nói: Server tra điểm bằng `Treasure.valueForType()` rồi mới cộng.

### Thầy hỏi “Sinh rương mới ở đâu?”

Mở:

```text
src/server/GameRoom.java
Ctrl+F: createReplacementTreasure
```

Nói: duyệt ô đường, loại Player/rương/vật cản rồi random trong danh sách hợp lệ.

### Thầy hỏi “BFS ở đâu?”

Mở:

```text
src/server/MapGenerator.java
Ctrl+F: isConnected
```

Nói: BFS từ tâm, map hợp lệ khi thăm được 41 ô đường.

### Thầy hỏi “Hai Client đồng bộ ở đâu?”

Mở:

```text
src/server/ServerManager.java
Ctrl+F: GameStateCodec.encode
```

Chỉ việc tạo một `Message state` rồi `sendToRoom(room, state)`.

### Thầy hỏi “Client cập nhật giao diện ở đâu?”

Mở:

```text
src/client/MainFrame.java
Ctrl+F: case Protocol.GAME_STATE
```

Sau đó mở:

```text
src/client/GamePanel.java
Ctrl+F: setGameState
```

## 4. Công thức phải nhớ

```text
cell = row * 7 + col
row = cell / 7
col = cell % 7
oppositeCell = 48 - cell
opposite(row,col) = (6-row, 6-col)
Manhattan = |row1-row2| + |col1-col2|
ô đường = 49 - 8 = 41
```

## 5. Các con số phải nhớ

| Nội dung | Giá trị |
|---|---:|
| Kích thước map | 7×7 |
| Tổng ô | 49 |
| Vật cản | 8 |
| Ô đường | 41 |
| Số lần thử map tối đa | 1000 |
| Rương ban đầu | 5 |
| NORMAL | 10 điểm |
| SILVER | 20 điểm |
| GOLD | 30 điểm |

## 6. Năm câu chốt

1. Client chỉ gửi ý định `MOVE`, không gửi tọa độ.
2. Server mới thay đổi `Position`, điểm và rương.
3. Map random phải đối xứng và vượt BFS liên thông.
4. MOVE, nhặt rương, respawn và broadcast cùng nằm dưới khóa Room.
5. `GAME_STATE` mới là snapshot chuẩn; event chỉ để thông báo.

## 7. Ranh giới trách nhiệm

Nếu câu hỏi đi sang phần khác, trả lời điểm giao tiếp rồi chuyển đúng phạm vi:

- Login/Socket: Vinh.
- Lobby/invite/tạo Room/rematch: Toàn.
- Timer/result/ranking/history/save: Tùng.

Ví dụ:

> Phần Toàn tạo Room và gọi `startGame()`. Từ `startGame()` trở đi, phần em sinh map và xử lý gameplay state.

> Phần em cộng match score khi nhặt rương. Khi trận kết thúc, phần Tùng dùng match score đó để xác định kết quả và rank.

