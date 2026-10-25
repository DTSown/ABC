# 00. Lộ trình học phần Sơn

## Mục tiêu cuối cùng

Sau khi học xong, bạn phải tự nói được luồng sau mà không nhìn tài liệu:

```text
WASD / phím mũi tên
→ GamePanel gửi MOVE|direction
→ ClientHandler nhận MOVE
→ ServerManager.move() xác thực phiên và tìm Room
→ khóa Room
→ GameRoom.handleMove() kiểm tra va chạm
→ GameRoom.collectTreasure() xử lý rương và điểm
→ GameStateCodec.encode()
→ GAME_STATE gửi tới cả hai Client
→ MainFrame decode
→ GamePanel cập nhật HUD
→ BoardPanel vẽ snapshot mới
```

## Vòng học thứ nhất — hiểu bức tranh

Thời gian khoảng 30–45 phút.

1. Đọc [01-tong-quan-va-luong-gameplay.md](01-tong-quan-va-luong-gameplay.md).
2. Mở `Protocol.java`, tìm `MOVE`, `GAME_STATE`, `TREASURE_COLLECTED`.
3. Mở `GameState.java`, xác định map, hai Player, treasures, time và status.
4. Tự vẽ luồng Client → Server → Client lên giấy.

Kết quả cần đạt:

- Biết Client chỉ gửi hướng.
- Biết Server mới đổi tọa độ và điểm.
- Biết `TREASURE_COLLECTED` khác `GAME_STATE`.

## Vòng học thứ hai — sinh bản đồ

Thời gian khoảng 45–60 phút.

1. Đọc [02-sinh-map-va-bfs.md](02-sinh-map-va-bfs.md).
2. Mở `MapGenerator.generate()` và đi từng bước theo comment.
3. Tự tính ba ví dụ:
   - cell `0` ↔ cell `48`;
   - cell `8` ↔ cell `40`;
   - cell `24` ↔ chính nó.
4. Mở `isConnected()` và tự mô tả queue của BFS.
5. Trả lời: tại sao cần BFS dù map đã đối xứng?

Kết quả cần đạt:

- Giải thích được 8 vật cản sinh từ 4 cặp.
- Giải thích được vì sao mọi ô đường đi tới được.
- Biết cách sinh hai spawn và năm rương công bằng.

## Vòng học thứ ba — MOVE và va chạm

Thời gian khoảng 60 phút.

1. Đọc [03-move-va-va-cham.md](03-move-va-va-cham.md).
2. Mở lần lượt:
   - `GamePanel.createMoveButton()`;
   - `ClientHandler.handleMessage()`;
   - `ServerManager.move()`;
   - `GameRoom.handleMove()`.
3. Với mỗi hàm, ghi ra:
   - dữ liệu vào;
   - điều kiện từ chối;
   - dữ liệu thay đổi;
   - message gửi tiếp.
4. Tự mô phỏng Player ở `(3,3)` gửi `UP`, `LEFT` và đi vào tường.

Kết quả cần đạt:

- Phân biệt kiểm tra hỗ trợ UI và validation có thẩm quyền ở Server.
- Chỉ đúng dòng Server thay đổi `Position`.
- Nói được ba loại va chạm.

## Vòng học thứ tư — rương và điểm

Thời gian khoảng 45 phút.

1. Đọc [04-nhat-ruong-va-diem.md](04-nhat-ruong-va-diem.md).
2. Mở `collectTreasure()` và `createReplacementTreasure()`.
3. Thuộc ánh xạ:
   - `NORMAL = 10`;
   - `SILVER = 20`;
   - `GOLD = 30`.
4. Giải thích vì sao phải loại Player, vật cản và rương cũ khi chọn ô respawn.
5. Giải thích giá trị trả về của `collectTreasure()`.

## Vòng học thứ năm — đồng bộ mạng

Thời gian khoảng 45–60 phút.

1. Đọc [05-dong-bo-game-state.md](05-dong-bo-game-state.md).
2. Viết tay cấu trúc bảy trường của `GAME_STATE`.
3. Mở `GameStateCodec.encode()` và `decode()` song song.
4. Chỉ ra vì sao hai Client nhận cùng state.
5. Chỉ ra vì sao `TREASURE_COLLECTED` không tự cộng điểm trên Client.

## Vòng học thứ sáu — xử lý đồng thời

Thời gian khoảng 30–45 phút.

1. Đọc [06-dong-thoi-va-server-authoritative.md](06-dong-thoi-va-server-authoritative.md).
2. Tự giải thích tình huống hai người cùng đi vào một ô.
3. Tự giải thích tình huống hai người cùng tranh một rương.
4. Chỉ khóa `synchronized(room)` trong `ServerManager.move()`.

## Vòng cuối — tập bảo vệ

1. Thực hành [07-kich-ban-demo-va-kiem-thu.md](07-kich-ban-demo-va-kiem-thu.md) ít nhất hai lần.
2. Tự trả lời [08-cau-hoi-phan-bien.md](08-cau-hoi-phan-bien.md), lần đầu xem đáp án, lần sau che đáp án.
3. Dùng [09-phao-code-va-bai-noi.md](09-phao-code-va-bai-noi.md) để tập mở file dưới 10 giây.

## Checklist tự đánh giá

- [ ] Tôi giải thích được `row`, `col` và chỉ số `cell`.
- [ ] Tôi giải thích được phép đối xứng `48 - cell`.
- [ ] Tôi giải thích được BFS và con số 41 ô đường.
- [ ] Tôi thuộc năm rương ban đầu.
- [ ] Tôi chỉ đúng nơi Client gửi `MOVE`.
- [ ] Tôi chỉ đúng nơi Server đổi `Position`.
- [ ] Tôi chỉ đúng nơi Server cộng điểm.
- [ ] Tôi giải thích được respawn rương.
- [ ] Tôi viết được cấu trúc `GAME_STATE`.
- [ ] Tôi giải thích được khóa Room và tranh rương đồng thời.
- [ ] Tôi phân biệt được event thông báo với snapshot trạng thái.
- [ ] Tôi demo được hai Client nhìn thấy cùng một vị trí.

