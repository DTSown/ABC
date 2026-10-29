# PHẦN SƠN — TỔNG HỢP KIẾN THỨC NHANH

> Đọc file này trước khi mở code. Mục tiêu là hiểu phần Sơn trong khoảng 10 phút và biết phải tìm đoạn nào khi giảng viên hỏi.

## 1. Phần Sơn làm gì?

Phần Sơn phụ trách **gameplay trong trận**:

1. Sinh bản đồ 7×7.
2. Chọn vị trí xuất phát cho hai người chơi.
3. Sinh các rương ban đầu.
4. Nhận hướng di chuyển từ Client.
5. Kiểm tra biên, vật cản và va chạm với đối thủ.
6. Nhặt rương, cộng điểm và sinh rương thay thế.
7. Gửi cùng một `GAME_STATE` cho hai Client.
8. Client dùng state đó để cập nhật HUD và vẽ bàn cờ.

Câu quan trọng nhất:

> **Client chỉ gửi ý định di chuyển; Server quyết định vị trí, điểm và rương.**

Đây là mô hình **Server authoritative**: Server là nguồn dữ liệu đúng duy nhất của trận đấu.

## 2. Toàn bộ luồng trong một hình

```text
Người chơi bấm WASD / phím mũi tên / nút điều hướng
                         |
                         v
GamePanel.createMoveButton(): chỉ tạo hướng UP/DOWN/LEFT/RIGHT
                         |
                         v
MainFrame gửi message MOVE|direction qua Socket
                         |
                         v
ClientHandler nhận MOVE và gọi ServerManager.move(this, direction)
                         |
                         v
ServerManager xác thực phiên + trạng thái PLAYING + tìm đúng Room
                         |
                         v
             synchronized (room)
                         |
                         v
GameRoom.handleMove(): kiểm tra hướng, biên, tường, đối thủ
                         |
              MOVE hợp lệ hay không?
                  /             \
                Không            Có
                 |               |
          gửi ERROR        cập nhật Position
                                 |
                                 v
                    GameRoom.collectTreasure()
                                 |
                       có rương hay không?
                         /              \
                      Không              Có
                        |                 |
                        |        cộng điểm + respawn rương
                        |        gửi TREASURE_COLLECTED
                        \_________________/
                                 |
                                 v
               encode và gửi cùng một GAME_STATE
                         cho cả hai Client
                                 |
                                 v
MainFrame decode -> GamePanel cập nhật HUD -> BoardPanel vẽ bàn cờ
```

## 3. Mở code theo đúng thứ tự

Trong VS Code hoặc IDE, nhấn `Ctrl+Shift+F` và tìm `PHẦN SƠN -` để thấy toàn bộ phần đã đánh dấu.

| Thứ tự | Từ khóa cần tìm | Ý nghĩa |
|---:|---|---|
| 1 | `PHẦN SƠN - SINH MAP BAN ĐẦU` | Điểm bắt đầu tạo map, spawn và rương |
| 2 | `PHẦN SƠN - THUẬT TOÁN SINH MAP` | Các bước random map đối xứng |
| 3 | `PHẦN SƠN - BFS KIỂM TRA LIÊN THÔNG` | Bảo đảm mọi ô đường đều đi tới được |
| 4 | `PHẦN SƠN - GỬI MOVE` | Client chỉ gửi tên hướng |
| 5 | `PHẦN SƠN - NHẬN MOVE QUA SOCKET` | Socket chuyển request sang ServerManager |
| 6 | `PHẦN SƠN - ĐIỀU PHỐI MOVE` | Luồng chính phía Server |
| 7 | `PHẦN SƠN - KIỂM TRA MOVE` | Kiểm tra biên, tường và đối thủ |
| 8 | `PHẦN SƠN - NHẶT RƯƠNG` | Cộng điểm và thay rương |
| 9 | `PHẦN SƠN - RESPAWN RƯƠNG` | Chọn ô trống cho rương mới |
| 10 | `PHẦN SƠN - ENCODE GAME_STATE` | Server đóng gói snapshot |
| 11 | `PHẦN SƠN - NHẬN GAME_STATE` | Client nhận và decode snapshot |
| 12 | `PHẦN SƠN - CẬP NHẬT GAME_STATE` | Cập nhật HUD, thời gian và Board |
| 13 | `PHẦN SƠN - RENDER SNAPSHOT` | Vẽ map, vật cản, rương và Player |

## 4. Sinh bản đồ hoạt động thế nào?

Code chính nằm trong `server/MapGenerator.java`.

### Dữ liệu cố định

- Map có kích thước `7 × 7` = 49 ô.
- `0` là đường đi.
- `1` là vật cản.
- Có 4 cặp vật cản đối xứng = 8 vật cản.
- Còn lại `49 - 8 = 41` ô đường.
- Tối đa thử sinh map 1000 lần.

### Đối xứng 180 độ

Một ô được đổi thành số phẳng:

```text
cell = row * 7 + col
row = cell / 7
col = cell % 7
```

Ô đối xứng:

```text
oppositeCell = 48 - cell
opposite(row, col) = (6 - row, 6 - col)
```

Ví dụ:

```text
(0,0) đối xứng với (6,6)
(1,2) đối xứng với (5,4)
(3,3) là ô trung tâm, đối xứng với chính nó
```

### Các bước sinh map

1. Tạo mảng `int[7][7]`, mặc định toàn bộ là đường `0`.
2. Chọn ngẫu nhiên 4 ô ở nửa đầu bản đồ.
3. Đặt vật cản tại mỗi ô và ô đối xứng của nó.
4. Chạy BFS kiểm tra 41 ô đường có liên thông không.
5. Chọn vị trí Player 1; Player 2 nằm ở ô đối xứng.
6. Yêu cầu khoảng cách Manhattan giữa hai spawn đủ xa.
7. Sinh hai rương thường, hai rương bạc và một rương vàng.
8. Reset điểm và số rương trong trận của hai Player về 0.
9. Trả về `GameState` có trạng thái `RUNNING`.

## 5. BFS dùng để làm gì?

BFS không tìm đường cho Player tự chạy. BFS chỉ dùng lúc **kiểm tra map vừa sinh**.

Thuật toán:

1. Bắt đầu từ ô trung tâm `(3,3)`.
2. Đưa ô đó vào hàng đợi `Queue`.
3. Lần lượt lấy một ô ra.
4. Xét bốn hàng xóm: trên, dưới, trái, phải.
5. Chỉ thêm ô nằm trong map, là đường và chưa thăm.
6. Đếm tổng số ô đi tới được.
7. Chỉ chấp nhận map nếu đếm được đúng 41 ô đường.

Tại sao phải kiểm tra?

> Map đối xứng chưa chắc liên thông. BFS loại các map có vùng đường bị cô lập.

## 6. MOVE được xử lý thế nào?

### Phía Client

`GamePanel` gom nút điều hướng, WASD và phím mũi tên vào cùng một `Action`.

Client chỉ gửi một trong bốn chuỗi:

```text
UP
DOWN
LEFT
RIGHT
```

Client **không gửi** tọa độ đích, điểm hoặc danh sách rương.

### Phía Server

`ServerManager.move()` kiểm tra:

1. Handler có phải đúng phiên đang đăng nhập không.
2. Handler có gắn với một `Player` không.
3. Player có đang ở trạng thái `PLAYING` không.
4. Player có thực sự thuộc Room đang xử lý không.

Sau đó Server khóa Room và gọi `GameRoom.handleMove()`.

### Công thức đổi vị trí

```text
UP    -> row - 1
DOWN  -> row + 1
LEFT  -> col - 1
RIGHT -> col + 1
```

Server từ chối bước đi nếu:

- hàng hoặc cột nằm ngoài `0..6`;
- `map[row][col] == 1`, tức là vật cản;
- ô đích đang có đối thủ;
- người gửi không thuộc Room hoặc không còn `PLAYING`.

Chỉ sau khi vượt qua tất cả kiểm tra, Server mới gọi:

```java
moving.setPosition(new Position(row, col));
```

## 7. Nhặt rương và cộng điểm

Sau một MOVE hợp lệ, `ServerManager` gọi `GameRoom.collectTreasure(player)`.

Điểm từng loại:

| Loại | Điểm |
|---|---:|
| `NORMAL` | 10 |
| `SILVER` | 20 |
| `GOLD` | 30 |

Luồng xử lý rương thưởng:

1. So sánh vị trí Player với từng rương.
2. Tính lại điểm bằng `Treasure.valueForType(type)`.
3. Tạo rương thay thế ở một ô hợp lệ.
4. Xóa rương cũ và thêm rương mới.
5. Cộng `matchScore`.
6. Tăng `collectedTreasure`.
7. Trả rương vừa nhặt cho `ServerManager` gửi event thông báo.

Rương mới không được nằm trên:

- vật cản;
- Player 1;
- Player 2;
- bất kỳ rương đang tồn tại nào.

### Phân biệt event và state

```text
TREASURE_COLLECTED = thông báo và âm thanh
GAME_STATE         = dữ liệu chính thức để cập nhật điểm, vị trí và rương
```

Client không tự cộng điểm khi nhận `TREASURE_COLLECTED`.

## 8. GAME_STATE chứa gì?

`GameState` là ảnh chụp toàn bộ Room tại một thời điểm:

```text
roomId
map 7×7
player1: username, position, matchScore, collectedTreasure
player2: username, position, matchScore, collectedTreasure
treasures
remainingTime
status
```

`GameStateCodec` chuyển object thành chuỗi gồm 7 phần:

```text
roomId ; map ; player1 ; player2 ; treasures ; time ; status
```

Khi decode, Client kiểm tra:

- payload có đúng 7 phần không;
- map có đúng 49 ký tự `0/1` không;
- vị trí có nằm trong map và trên ô đường không;
- hai Player có trùng vị trí hoặc username không;
- ID rương có bị trùng không;
- giá trị rương có đúng với loại không;
- điểm, số rương và thời gian có âm không.

Chỉ dữ liệu hợp lệ mới được chuyển sang giao diện.

## 9. Vì sao hai Client luôn đồng bộ?

Trong `ServerManager.move()`:

```java
synchronized (room) {
    // kiểm tra MOVE
    // cập nhật Position
    // nhặt rương
    // cộng điểm
    // respawn rương
    // encode GAME_STATE
    // gửi cho cả hai Client
}
```

Ý nghĩa của khóa Room:

- Hai MOVE đồng thời được xử lý tuần tự.
- Hai người không thể cùng ăn một rương.
- Không Client nào nhận state đang cập nhật dở.
- Event và snapshot được phát theo đúng thứ tự.
- Hai Client nhận cùng payload `GAME_STATE`.

## 10. Client hiển thị state thế nào?

1. `MainFrame` nhận message `GAME_STATE`.
2. `GameStateCodec.decode()` kiểm tra và tạo `GameState`.
3. `GamePanel.setGameState()` lưu state mới.
4. Hai HUD lấy điểm và số rương từ hai Player.
5. Đồng hồ lấy `remainingTime` từ Server.
6. `BoardPanel.setSnapshot()` nhận snapshot.
7. `BoardPanel.paintComponent()` vẽ theo thứ tự:
   - nền 49 ô;
   - vật cản;
   - rương;
   - Player 1 và Player 2.

`BoardPanel` không tự di chuyển nhân vật và không tự sinh rương.

## 11. Phần bẫy trong source hiện tại

Source hiện tại có mở rộng bẫy nằm trên cùng luồng gameplay:

- một số rương được bí mật đánh dấu là bẫy ở Server;
- dính bẫy có thể bị trừ tối đa 10 điểm;
- Player bị choáng 2 giây;
- Server phát `TRAP_TRIGGERED`;
- Client chỉ hiện hiệu ứng và tạm khóa thao tác;
- quyền quyết định bẫy và điểm vẫn nằm ở Server.

Khi thuyết trình, nên trình bày phần được giao trước: map, MOVE, va chạm, rương thưởng, điểm và `GAME_STATE`. Chỉ nói bẫy khi được hỏi về source mở rộng.

## 12. Ranh giới với các thành viên khác

| Thành viên | Phần chính |
|---|---|
| Vinh | Login, đăng ký, Socket và kết nối |
| Toàn | Lobby, online, lời mời, tạo Room và tái đấu |
| **Sơn** | **Map, MOVE, va chạm, rương, điểm trong trận và GAME_STATE** |
| Tùng | Timer, kết quả, rank, history và lưu dữ liệu |

Điểm bàn giao:

- Toàn tạo Room và gọi `startGame()`; từ đây Sơn sinh gameplay state.
- Vinh nhận message qua Socket; nhánh `MOVE` chuyển vào luồng của Sơn.
- Sơn tạo `matchScore`; khi trận kết thúc, Tùng dùng điểm đó để tạo kết quả.

## 13. Các lỗi nói sai cần tránh

- Không nói Client tự thay đổi tọa độ.
- Không nói Client tự cộng điểm.
- Không nói `TREASURE_COLLECTED` là state chính thức.
- Không nói BFS dùng để tự động tìm đường cho Player.
- Không nói map đối xứng thì tự động liên thông.
- Không nói hệ thống ghép cặp ngẫu nhiên; người chơi vào Room qua lời mời.
- Không nhận timer, kết quả, rank và history là phần chính của Sơn.

## 14. Bài nói 60 giây

> Phần em phụ trách là gameplay trong trận. Khi Room bắt đầu, Server sinh map 7×7 với tám vật cản đối xứng, sau đó dùng BFS để bảo đảm 41 ô đường liên thông. Hai người chơi có vị trí xuất phát đối xứng và Server tạo năm rương ban đầu. Khi người dùng bấm phím, Client chỉ gửi hướng MOVE, không gửi tọa độ hay điểm. Server xác thực người gửi, khóa Room, kiểm tra biên, vật cản và đối thủ rồi mới cập nhật Position. Sau MOVE hợp lệ, Server kiểm tra rương, tự cộng 10, 20 hoặc 30 điểm và sinh rương thay thế ở ô trống. Cuối cùng Server encode một GAME_STATE và gửi cùng state cho hai Client. Client chỉ decode, cập nhật HUD và vẽ bàn cờ. Vì vậy Server là nguồn sự thật và hai màn hình luôn được đồng bộ.

## 15. Câu hỏi nhanh và câu trả lời

### Tại sao Client chỉ gửi hướng?

Để Client không thể tự chọn tọa độ đích, đi xuyên tường hoặc sửa điểm.

### Tại sao cần BFS?

Để loại map có vùng đường bị cô lập, bảo đảm toàn bộ 41 ô đường liên thông.

### Tại sao vật cản và spawn phải đối xứng?

Để hai phía có điều kiện xuất phát công bằng về mặt hình học.

### Tại sao phải khóa Room?

Để MOVE, nhặt rương, cộng điểm, respawn và broadcast trở thành một chuỗi xử lý không bị request khác chen ngang.

### Điểm được cộng ở đâu?

Ở Server trong `GameRoom.collectTreasure()`, dùng `Treasure.valueForType()`.

### Hai Client đồng bộ ở đâu?

Ở cuối `ServerManager.move()`: Server encode state sau cập nhật rồi `sendToRoom()` cho đúng hai thành viên.

### Client vẽ bản đồ ở đâu?

Trong `BoardPanel.paintComponent()`, từ snapshot đã nhận qua `setSnapshot()`.

## 16. Phao nhớ cuối cùng

```text
MAP:      7×7 = 49 ô, 8 vật cản, 41 ô đường
RƯƠNG:    NORMAL 10, SILVER 20, GOLD 30
INPUT:    Client chỉ gửi UP/DOWN/LEFT/RIGHT
SERVER:   kiểm tra -> cập nhật -> nhặt rương -> respawn -> broadcast
ĐỒNG BỘ:  một GAME_STATE gửi cho cả hai Client
AN TOÀN:  mọi thay đổi gameplay nằm dưới khóa Room
```

Nếu chỉ nhớ năm câu, hãy nhớ:

1. Client gửi ý định, Server quyết định kết quả.
2. Map đối xứng vẫn phải chạy BFS để kiểm tra liên thông.
3. Server kiểm tra biên, vật cản và đối thủ trước khi đổi Position.
4. Server tự tính điểm và tự sinh rương thay thế.
5. Hai Client cùng nhận một `GAME_STATE`, nên luôn đồng bộ.
