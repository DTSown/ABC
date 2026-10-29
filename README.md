# Tài liệu phần Sơn — Gameplay và đồng bộ trạng thái

Đây là bộ tài liệu ôn bảo vệ dành riêng cho **Sơn**, bám theo bảng phân công:

> Bản đồ, di chuyển, va chạm, nhặt rương, tăng điểm và đồng bộ `GAME_STATE`.

## Phạm vi phải nắm chắc

1. Client hiển thị bản đồ 7×7 và nhân vật từ snapshot Server.
2. Server sinh bản đồ ngẫu nhiên, đối xứng và liên thông.
3. Hai vị trí xuất phát đối xứng, đủ xa nhau.
4. Năm rương ban đầu: hai thường, hai bạc và một vàng.
5. WASD, phím mũi tên và nút điều hướng chỉ gửi `MOVE`.
6. Server xác thực người gửi, Room và trạng thái `PLAYING`.
7. Server kiểm tra biên, vật cản và vị trí đối thủ.
8. Server nhặt rương, cộng 10/20/30 điểm và sinh rương thay thế.
9. Server gửi cùng `GAME_STATE` cho hai Client.
10. Khóa Room giúp xử lý đúng hai lệnh di chuyển hoặc tranh rương đồng thời.

## Những phần không phải trách nhiệm chính của Sơn

- Vinh: đăng nhập, đăng ký, Socket, lỗi mạng và xác thực.
- Toàn: Lobby, online, lời mời, tạo Room và tái đấu.
- Tùng: timer, kết quả, rank, history và persistence.

Sơn vẫn phải biết **điểm giao tiếp** với các phần trên, nhưng không nên nhận đó là phần chính của mình.

## Thứ tự đọc

Nếu bạn chỉ có **1 giờ**, đọc thẳng [11 — Hiểu toàn bộ luồng và code trong 60 phút](11-hieu-toan-bo-trong-60-phut.md), vừa đọc vừa mở các method trong bảng “Phao mở code”.

Nếu học kỹ để phản biện, đọc bộ đầy đủ theo thứ tự dưới đây:

1. [00 — Lộ trình học](00-lo-trinh-hoc.md)
2. [01 — Tổng quan và luồng end-to-end](01-tong-quan-va-luong-gameplay.md)
3. [02 — Sinh bản đồ, đối xứng và BFS](02-sinh-map-va-bfs.md)
4. [03 — MOVE và xử lý va chạm](03-move-va-va-cham.md)
5. [04 — Nhặt rương, cộng điểm và respawn](04-nhat-ruong-va-diem.md)
6. [05 — Đồng bộ GAME_STATE](05-dong-bo-game-state.md)
7. [06 — Đồng thời và Server authoritative](06-dong-thoi-va-server-authoritative.md)
8. [07 — Kịch bản demo và kiểm thử](07-kich-ban-demo-va-kiem-thu.md)
9. [08 — Câu hỏi phản biện](08-cau-hoi-phan-bien.md)
10. [09 — Phao code và bài nói](09-phao-code-va-bai-noi.md)

## Các file source cần mở được trong 10 giây

| Mục đích | File / method |
|---|---|
| Vẽ bản đồ | `client/BoardPanel.java`: `setSnapshot()`, `paintComponent()` |
| Gửi hướng di chuyển | `client/GamePanel.java`: `createMoveButton()` |
| Nhận state và cập nhật HUD | `client/GamePanel.java`: `setGameState()` |
| Nhận message từ Server | `client/MainFrame.java`: nhánh `GAME_STATE`, `TREASURE_COLLECTED` |
| Message MOVE đến Server | `server/ClientHandler.java`: nhánh `Protocol.MOVE` |
| Điều phối toàn bộ MOVE | `server/ServerManager.java`: `move()` |
| Kiểm tra va chạm | `server/GameRoom.java`: `handleMove()` |
| Nhặt và sinh lại rương | `server/GameRoom.java`: `collectTreasure()`, `createReplacementTreasure()` |
| Sinh map ban đầu | `server/MapGenerator.java`: `generate()` |
| BFS | `server/MapGenerator.java`: `isConnected()` |
| Điểm rương | `model/Treasure.java`: `valueForType()` |
| Snapshot trận | `model/GameState.java` |
| Encode/decode snapshot | `common/GameStateCodec.java` |

## Hai điều dễ nói nhầm

### Không có random matchmaking

Map được sinh ngẫu nhiên, nhưng hai người được ghép bằng lời mời do phần Toàn xử lý. Không được nói dự án có “tìm đối thủ ngẫu nhiên”.

### Source hiện tại có bẫy/choáng

Bảng phân công gốc ghi bản ZIP chưa có stun, nhưng repository hiện tại đã có `TRAP_TRIGGERED` và choáng. Khi bảo vệ phần được giao, hãy trình bày luồng rương thưởng và `GAME_STATE` trước. Nếu giảng viên hỏi source hiện tại, nói rõ bẫy là phần mở rộng nằm cùng đường xử lý gameplay.

## Câu chốt phần Sơn

> Client chỉ gửi ý định di chuyển, không gửi tọa độ hay điểm. Server kiểm tra luật, cập nhật trạng thái dưới khóa Room rồi gửi cùng một `GAME_STATE` cho hai Client. Vì vậy Server là nguồn sự thật và hai màn hình luôn được đồng bộ.

# ltm
"# ABC" 
