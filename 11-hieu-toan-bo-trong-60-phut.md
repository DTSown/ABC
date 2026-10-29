# Phần Sơn — hiểu toàn bộ luồng và code trong 60 phút

> Mục tiêu: sau một giờ, bạn có thể tự lần từ một lần bấm phím ở Client đến lúc cả hai màn hình vẽ state mới, đồng thời giải thích được sinh map, va chạm, rương, bẫy, đồng bộ và kết thúc trận.

## 0–5 phút: giữ đúng mô hình trong đầu

Câu chốt của toàn bộ phần này:

> **Client gửi ý định; Server kiểm tra luật, thay đổi state và phát kết quả.**

```text
INPUT CLIENT
WASD / phím mũi tên / nút bấm
        |
        v
MOVE|UP/DOWN/LEFT/RIGHT
        |
        v
SERVER AUTHORITATIVE
xác thực phiên -> khóa -> kiểm tra luật -> cập nhật GameState
        |
        +----> event: TREASURE_COLLECTED hoặc TRAP_TRIGGERED
        |
        `----> snapshot: GAME_STATE
                         |
                         v
CLIENT PRESENTATION
decode -> HUD -> BoardPanel.repaint()
```

Ba khái niệm không được trộn:

| Khái niệm | Vai trò |
|---|---|
| `GameState` | Ảnh chụp chính thức của một Room: map, hai Player, rương, thời gian, status |
| Event | Báo một việc vừa xảy ra để hiện chữ/âm thanh/hiệu ứng |
| UI state | Dữ liệu trình bày tạm thời như notice, animation choáng; không quyết định điểm/vị trí |

## 5–12 phút: các object và nơi giữ sự thật

### Mô hình dữ liệu

`Position`

- `row`: tăng khi `DOWN`, giảm khi `UP`.
- `col`: tăng khi `RIGHT`, giảm khi `LEFT`.
- Miền hợp lệ là `0..6`.

`Player`

- Dữ liệu trong trận: `position`, `matchScore`, `collectedTreasure`, `status`.
- Dữ liệu dài hạn: `totalRankPoint`, `wins`, `totalMatches`, `totalTreasureCollected`.
- Khi mở trận mới chỉ dữ liệu trong trận được reset.

`Treasure`

| Type | Điểm |
|---|---:|
| `NORMAL` | 10 |
| `SILVER` | 20 |
| `GOLD` | 30 |

Server luôn tính lại điểm bằng `Treasure.valueForType(type)`, không tin một giá trị điểm do Client khai.

`GameState`

```text
roomId
map[7][7]
player1
player2
treasures
remainingTime
status: WAITING / RUNNING / FINISHED
```

### Ai sở hữu gì?

- `GameRoom.gameState` là state có thẩm quyền ở Server.
- `GamePanel.currentState` là bản đã decode để hiển thị ở Client.
- `BoardPanel` chỉ đọc snapshot và vẽ; không tự tính bước đi, điểm hay rương mới.
- `TIME_UPDATE` là ngoại lệ nhỏ: Client cập nhật `remainingTime` trên bản local giữa hai full snapshot. Nó vẫn dùng số do Server gửi, không tự đếm để quyết định kết quả.

Mở nhanh: `model/GameState.java`, `model/Player.java`, `model/Treasure.java`.

## 12–20 phút: vòng đời bắt đầu trận và sinh map

### Từ lời mời đến state đầu tiên

```text
Client nhận lời mời
-> ServerManager.acceptInvite()
-> createGameRoom(player1, player2)
-> new GameRoom(...)
-> room.startGame(previousMap)
-> MapGenerator.generate(...)
-> gắn GameState RUNNING
-> Player status = PLAYING
-> startRoom(room)
-> gửi GAME_START
-> gửi GAME_STATE đầu tiên
-> khởi động timer mỗi giây
```

`GAME_START` chỉ báo tên hai người để chuyển giao diện. `GAME_STATE` sau đó mới chứa map và gameplay state thật.

### Sinh map trong `MapGenerator.generate()`

Các hằng số phải nhớ:

```text
7 x 7 = 49 ô
8 vật cản
41 ô đường
5 rương ban đầu
180 giây mặc định (ServerManager có thể thay bằng thời lượng cấu hình)
```

Hệ tọa độ phẳng:

```text
cell = row * 7 + col
row  = cell / 7
col  = cell % 7
opposite(cell) = 48 - cell
opposite(row, col) = (6-row, 6-col)
```

Thuật toán:

1. Tạo map toàn số `0`.
2. Xáo các cell `0..23`, lấy 4 cell.
3. Đặt tường ở 4 cell đó và 4 cell đối xứng: tổng 8 tường.
4. Chạy BFS từ tâm cell `24`, tức `(3,3)`.
5. Chỉ nhận map nếu BFS thăm đủ `49 - 8 = 41` ô đường.
6. Chọn spawn P1 trên đường sao cho khoảng cách Manhattan đến ô đối xứng ít nhất 5; P2 ở ô đối xứng.
7. Chọn hai cặp vị trí rương cách mỗi spawn ít nhất 2 bước.
8. Tạo `T1,T2` là NORMAL, `T3,T4` là SILVER, `T5` là GOLD ở tâm.
9. Reset vị trí, điểm và số rương trong trận; trả state `RUNNING`.

Vì sao cần cả đối xứng lẫn BFS?

- Đối xứng tạo công bằng hình học.
- Đối xứng không chứng minh liên thông.
- BFS chứng minh mọi ô đường thuộc cùng một vùng đi được.

Giới hạn 1000 lần giúp tránh vòng random vô hạn. Tái đấu còn có vòng sinh lại để map mới không trùng map cũ.

## 20–35 phút: lần trọn một lệnh MOVE

Đây là luồng quan trọng nhất. Đọc code đúng thứ tự sau.

### 1. Client tạo ý định

Trong `GamePanel.createMoveButton()`:

- Mỗi hướng có một Swing `Action`.
- Nút, WASD và phím mũi tên cùng trỏ vào Action đó.
- `canMove()` chỉ là guard UX.
- Khi hợp lệ, callback chỉ nhận `UP`, `DOWN`, `LEFT` hoặc `RIGHT`.
- Client không tự gọi `setPosition()`.

Trong `MainFrame.createGamePanel()`, callback được nối thành:

```java
new Message(Protocol.MOVE, direction)
```

`Message.encode()` tạo một dòng như `MOVE|UP`. `ClientConnection` ghi dòng UTF-8 xuống TCP và cấm ký tự xuống dòng để tránh chèn thêm message.

### 2. Server nhận message

Mỗi `ClientHandler` đọc Socket theo từng dòng:

```text
readLine()
-> Message.decode()
-> case MOVE
-> serverManager.move(this, message.getData())
```

Server nhận cả object handler `this`, nên danh tính đến từ phiên kết nối đã login, không đến từ username tùy ý trong payload.

Client gửi các message chỉ dành cho Server như `GAME_STATE`, `TIME_UPDATE`, `TREASURE_COLLECTED` hoặc `TRAP_TRIGGERED` sẽ bị trả `ERROR`.

### 3. `ServerManager.move()` xác thực và khóa

Thứ tự kiểm tra:

1. Handler phải đúng object đang nằm trong `clientsByUsername`.
2. Handler phải có `Player`.
3. Player phải đang `PLAYING`.
4. Tìm Room chứa đúng object Player đó bằng so sánh identity.
5. Vào `synchronized(room)`.

Lưu ý về khóa thực tế:

- Bản thân `move()` là `synchronized`, nên giữ khóa `ServerManager` trước.
- Sau đó code giữ thêm khóa Room.
- Các method chính của `GameRoom` cũng `synchronized`; Java monitor là reentrant nên lời gọi lồng nhau không tự deadlock.
- Thứ tự khóa nhất quán là manager trước, room sau.

### 4. `GameRoom.handleMove()` áp luật

Hàm trả:

- `null`: MOVE thành công, vị trí đã đổi.
- chuỗi lỗi: MOVE bị từ chối, state không đổi.
- `MOVE_STUNNED`: đang choáng, Server im lặng bỏ qua request.

Server tìm `moving` và `opponent` từ username đã xác thực, rồi tính ô đích:

```text
UP    -> row - 1
DOWN  -> row + 1
LEFT  -> col - 1
RIGHT -> col + 1
```

Ba va chạm bị chặn:

1. Ngoài biên `0..6`.
2. Vật cản: `map[row][col] == 1`.
3. Ô đang có đối thủ.

Chỉ sau tất cả kiểm tra mới chạy:

```java
moving.setPosition(new Position(row, col));
```

### 5. Kết quả của MOVE

Nếu MOVE sai:

```text
ERROR chỉ gửi cho người bấm
Position/điểm/rương không đổi
không phát GAME_STATE giả mới
```

Nếu MOVE đúng:

```text
Position đổi
-> collectTreasure(player)
-> phát event nếu cần
-> encode GameState sau cùng
-> sendToRoom() cho đúng hai thành viên
```

## 35–43 phút: rương thưởng, respawn và bẫy

### Rương thưởng bình thường

`collectTreasure()` so sánh tọa độ Player với từng rương. Nếu trùng:

1. Tính lại điểm từ `type`.
2. Tạo rương thay thế trước khi xóa rương cũ.
3. Xóa rương cũ, thêm rương mới.
4. Cộng `matchScore`.
5. Tăng `collectedTreasure`.
6. Trả rương cũ cho `ServerManager` phát `TREASURE_COLLECTED`.

Tạo rương mới trước giúp nếu không còn ô hợp lệ và method ném lỗi, state chưa bị mất rương cũ.

`createReplacementTreasure()` đánh dấu hai Player và toàn bộ rương hiện tại là occupied, liệt kê mọi ô đường còn trống, sau đó chọn ngẫu nhiên:

- một trong ba type;
- một vị trí hợp lệ;
- ID UUID mới.

Do đó luôn duy trì 5 rương. Sau thời điểm ban đầu, phân bố không bắt buộc còn `2 NORMAL + 2 SILVER + 1 GOLD` vì type respawn được random lại.

### Event không phải state

```text
TREASURE_COLLECTED|username;type;value;totalScore
```

Event dùng cho notice và âm thanh. `GamePanel.showTreasureCollected()` không cộng điểm và không xóa rương. HUD/board chỉ đổi khi `GAME_STATE` tiếp theo tới.

### Nhánh bẫy của source hiện tại

- Khi bắt đầu trận, đúng một trong 5 object rương được bí mật đánh dấu là bẫy.
- Danh sách bẫy là identity set ở Server và không được encode trong `GAME_STATE`.
- Mỗi rương respawn có xác suất 20% trở thành bẫy.
- Chạm bẫy vẫn xóa và respawn rương.
- Không cộng điểm, không tăng số rương; trừ `min(10, matchScore)`.
- Player bị Server chặn MOVE trong 2000 ms.
- Server phát `TRAP_TRIGGERED`, rồi vẫn phát `GAME_STATE` mới.
- Trong nhánh này `collectTreasure()` trả `null`; event bẫy được lấy riêng bằng `pollTrapEvent()`.

Client hiển thị sao quay và khóa nút tạm thời để UX khớp Server. Quyền từ chối MOVE thật vẫn ở Server, nên sửa UI không phá được luật choáng.

## 43–51 phút: wire format, đồng bộ và render

### Hai tầng đóng gói

Tầng message chung:

```text
TYPE|DATA
```

Tầng payload `GAME_STATE` gồm đúng 7 phần:

```text
roomId ; map ; player1 ; player2 ; treasures ; remainingTime ; status
```

Chi tiết:

```text
map:      49 ký tự 0/1 theo thứ tự từng hàng
player:   URLencodedUsername,row,col,matchScore,collectedTreasure
treasure: id,type,value,row,col
nhiều treasure ngăn bằng '/'
```

`GameStateCodec.decode()` kiểm tra:

- đủ 7 phần, có room ID và status;
- map đúng 49 ký tự `0/1`;
- số không âm;
- vị trí nằm trong map, trên đường và không trùng nhau;
- hai username khác nhau;
- ID rương duy nhất;
- value khớp type.

Codec không tự chứng minh lại map có đúng 8 tường, liên thông hay đúng 5 rương. Các bất biến đó do Server tạo state bảo đảm; decode tập trung chặn payload hỏng trước khi render.

### Vì sao hai Client thấy cùng kết quả?

Trong cùng critical section, Server:

1. hoàn tất mọi mutation;
2. gọi `GameStateCodec.encode()` một lần;
3. tạo một `Message GAME_STATE`;
4. `sendToRoom()` gửi message đó tới hai handler còn gắn đúng object Player.

Hai Client khỏe mạnh vì vậy nhận cùng payload và cùng thứ tự event → snapshot. TCP giữ thứ tự byte trên từng kết nối. Nếu một Client đã mất kết nối thì không thể nói “luôn nhận”; Server sẽ xử lý disconnect và kết thúc trận.

### Client nhận và vẽ

```text
ClientConnection.listenServer()
-> chuyển callback lên Swing EDT
-> MainFrame.onMessage()
-> case GAME_STATE
-> GameStateCodec.decode(data)
-> GamePanel.setGameState(state)
-> cập nhật hai HUD + room + time
-> BoardPanel.setSnapshot(state, username)
-> repaint()
-> paintComponent(): nền -> tường -> rương -> Player -> hiệu ứng
```

Không có client-side prediction: nhân vật chỉ nhảy sang ô mới sau khi snapshot Server quay về.

## 51–55 phút: timer và kết thúc trận

`startRoom()` khởi động một scheduled task mỗi giây. `tickRoom()`:

1. giữ khóa manager, rồi khóa room;
2. giảm `remainingTime` nhưng không dưới 0;
3. gửi `TIME_UPDATE` cho hai Client;
4. khi bằng 0, gọi `finishRoom(..., TIME_UP)`.

Kết thúc do hết giờ, thoát hoặc disconnect đều hội tụ vào `finishRoom()`:

```text
GameRoom.finishGame()
-> tạo GameResult
-> cập nhật thống kê dài hạn
-> endGame(): cancel timer, clear trap, status FINISHED
-> Server gửi GAME_STATE cuối
-> Server gửi GAME_OVER
-> cleanup Room, Player trở lại FREE
```

Thoát/disconnect dùng người bỏ cuộc để xác định người thắng; hết giờ dùng điểm trong state.

## 55–60 phút: tự kiểm tra không nhìn tài liệu

Hãy nói thành tiếng mười câu trả lời sau:

1. Tại sao Client chỉ gửi direction thay vì tọa độ?
2. Công thức đổi `(row,col)` sang cell và ô đối xứng là gì?
3. Vì sao 8 tường để lại 41 ô đường, và BFS chứng minh điều gì?
4. Dòng code nào thực sự đổi vị trí Player?
5. Ba loại va chạm bị chặn là gì?
6. Điểm rương được quyết định ở đâu?
7. Vì sao respawn không thể đè lên Player hoặc rương khác?
8. `TREASURE_COLLECTED` khác `GAME_STATE` thế nào?
9. Hai MOVE đồng thời tranh một ô/rương được tuần tự hóa ở đâu?
10. Khi thời gian về 0, thứ tự `TIME_UPDATE`, state cuối và `GAME_OVER` là gì?

Nếu trả lời được mạch lạc, bạn đã nắm luồng. Nếu vấp ở câu nào, chỉ đọc lại đúng khung thời gian tương ứng, không cần đọc lại toàn bộ.

## Phao mở code trong lúc bảo vệ

| Câu hỏi | File → method |
|---|---|
| Sinh map ở đâu? | `src/server/MapGenerator.java` → `generate()` |
| BFS ở đâu? | `src/server/MapGenerator.java` → `isConnected()` |
| Bắt đầu Room ở đâu? | `src/server/ServerManager.java` → `createGameRoom()`, `startRoom()` |
| Client gửi MOVE ở đâu? | `src/client/GamePanel.java` → `createMoveButton()` |
| Socket Server nhận MOVE ở đâu? | `src/server/ClientHandler.java` → `handleMessage()` |
| Xác thực và điều phối MOVE ở đâu? | `src/server/ServerManager.java` → `move()` |
| Chặn biên/tường/đối thủ ở đâu? | `src/server/GameRoom.java` → `handleMove()` |
| Cộng điểm và xử lý bẫy ở đâu? | `src/server/GameRoom.java` → `collectTreasure()` |
| Sinh rương thay thế ở đâu? | `src/server/GameRoom.java` → `createReplacementTreasure()` |
| Encode/decode state ở đâu? | `src/common/GameStateCodec.java` |
| Client nhận state ở đâu? | `src/client/MainFrame.java` → `onMessage()` |
| HUD/board cập nhật ở đâu? | `src/client/GamePanel.java` → `setGameState()` |
| Vẽ board ở đâu? | `src/client/BoardPanel.java` → `paintComponent()` |
| Timer/kết thúc ở đâu? | `src/server/ServerManager.java` → `tickRoom()`, `finishRoom()` |

## Một câu tóm tắt cuối

> Room bắt đầu bằng một `GameState` do Server sinh; Client chỉ gửi hướng, Server tuần tự hóa request, áp luật, xử lý rương/bẫy rồi broadcast snapshot; Client validate snapshot và chỉ dùng nó để cập nhật HUD, bàn cờ, trong khi timer và kết thúc trận cũng do Server điều phối.
