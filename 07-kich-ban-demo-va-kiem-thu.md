# 07. Kịch bản demo và kiểm thử phần Sơn

## 1. Chuẩn bị

1. Build dự án:

```powershell
.\build.ps1
```

2. Chạy Server:

```powershell
java -jar .\dist\TreasureRushServer.jar
```

3. Chạy hai Client:

```powershell
java -jar .\dist\TreasureRushClient.jar
```

4. Kết nối `127.0.0.1:5000` nếu cùng máy.
5. Đăng nhập hai tài khoản và nhờ phần Toàn đưa hai người vào trận.

## 2. Các tab source mở sẵn

1. `src/client/GamePanel.java`
2. `src/server/ServerManager.java`
3. `src/server/GameRoom.java`
4. `src/server/MapGenerator.java`
5. `src/common/GameStateCodec.java`
6. `src/client/BoardPanel.java`

## 3. Kịch bản nói và thao tác

### Bước 1 — Chỉ bản đồ

Thao tác: đặt hai cửa sổ Client cạnh nhau.

Lời nói:

> Bản đồ 7×7 được sinh duy nhất ở Server. Tám vật cản là bốn cặp đối xứng 180 độ. Server chạy BFS để bảo đảm mọi ô đường liên thông, sau đó gửi cùng GAME_STATE cho hai Client nên hai bên nhìn thấy cùng map.

Mở code: `MapGenerator.generate()` và `isConnected()`.

### Bước 2 — Chỉ năm rương

Lời nói:

> Ban đầu có hai rương thường, hai rương bạc theo cặp đối xứng và một rương vàng ở tâm. Điểm tương ứng là 10, 20 và 30.

Mở code: `MapGenerator.addPair()` và `Treasure.valueForType()`.

### Bước 3 — Di chuyển hợp lệ

Thao tác: bấm một phím để đi vào ô đường.

Lời nói:

> Client chỉ gửi direction trong message MOVE. Server xác định Player từ handler, kiểm tra Room và luật di chuyển, sau đó gửi snapshot mới cho cả hai. Vì vậy cửa sổ đối thủ cũng thấy vị trí thay đổi.

Mở code: `GamePanel.createMoveButton()`, `ServerManager.move()`, `GameRoom.handleMove()`.

### Bước 4 — Đâm vào tường hoặc biên

Thao tác: cố di chuyển vào vật cản hoặc mép bản đồ.

Lời nói:

> Server tính ô đích rồi kiểm tra biên và map. Nếu ô là vật cản hoặc ngoài 0 đến 6, Position không thay đổi. Client không thể tự đi xuyên tường.

### Bước 5 — Nhặt rương

Thao tác: di chuyển đến một rương.

Lời nói:

> Sau MOVE hợp lệ, Server so Position của Player với từng rương. Với rương thưởng, Server tự tính điểm từ type, tăng count, xóa rương cũ và sinh rương mới ở một ô hợp lệ. TREASURE_COLLECTED chỉ hiện thông báo; GAME_STATE sau đó mới cập nhật HUD và Board.

Mở code: `GameRoom.collectTreasure()` và `createReplacementTreasure()`.

### Bước 6 — Chứng minh đồng bộ

Thao tác: so sánh hai cửa sổ sau vài bước đi và nhặt rương.

Lời nói:

> Server encode một GameState sau cùng rồi gửi cùng payload cho hai thành viên Room. Hai Client cùng decode bằng GameStateCodec nên vị trí, điểm và rương giống nhau.

## 4. Nếu demo khó gặp rương

- Mở map và chọn đường ngắn nhất tới rương gần nhất.
- Có thể chỉ vào HUD trước và sau khi nhặt.
- Nếu gặp bẫy trong source hiện tại, giải thích đây là phần mở rộng gameplay; tiếp tục tới một rương thưởng để demo cộng điểm.

## 5. Nếu không chạy được demo

Trình bày trace bằng code:

```text
GamePanel.createMoveButton
→ moveSender.accept("UP")
→ Message(MOVE, "UP")
→ ClientHandler case MOVE
→ ServerManager.move
→ GameRoom.handleMove
→ GameRoom.collectTreasure
→ GameStateCodec.encode
→ sendToRoom
→ MainFrame case GAME_STATE
→ GamePanel.setGameState
→ BoardPanel.setSnapshot
```

## 6. Kiểm thử liên quan trực tiếp

### Chạy toàn bộ

```powershell
.\test.ps1
```

### Test cần nhớ

| Test | Nội dung |
|---|---|
| `MapGenerationTest` | đối xứng, liên thông, spawn, rương và codec |
| `MoveIntegrationTest` | hướng, biên, tường, va chạm và đồng bộ TCP |
| `TreasureCollectionTest` | điểm, count, respawn, ID và tranh rương |
| `GamePanelRenderTest` | input, HUD, map và render snapshot |

## 7. Các trường hợp phải thử miệng

1. MOVE hợp lệ trên ô trống.
2. MOVE ra ngoài biên.
3. MOVE vào tường.
4. MOVE vào đối thủ.
5. Direction không hợp lệ.
6. Người chưa login gửi MOVE.
7. Người FREE gửi MOVE.
8. Người không thuộc Room gửi MOVE.
9. Nhặt NORMAL/SILVER/GOLD.
10. Hai người tranh cùng rương.
11. Hai MOVE xảy ra đồng thời.
12. Client tự gửi GAME_STATE giả.

## 8. Checklist trước khi demo

- [ ] Hai Client đã vào cùng một Room.
- [ ] Cửa sổ không che nhau.
- [ ] Biết vị trí một vật cản để demo.
- [ ] Biết đường tới một rương thưởng.
- [ ] Mở sẵn sáu file source.
- [ ] Đã chạy test ít nhất một lần.
- [ ] Biết cách trả lời nếu gặp bẫy.

