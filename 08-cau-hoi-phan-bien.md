# 08. Câu hỏi phản biện phần Sơn

Mỗi câu có đáp án ngắn để trả lời ngay và phần giải thích để hiểu sâu.

## A. Bản đồ

### 1. Ai sinh bản đồ, Client hay Server?

**Trả lời ngắn:** Server.

**Giải thích:** `GameRoom.startGame()` gọi `MapGenerator.generate()`. Client chỉ nhận map trong `GAME_STATE` rồi vẽ bằng `BoardPanel`.

### 2. Bản đồ có kích thước bao nhiêu?

**Trả lời ngắn:** 7×7, tức 49 ô.

**Giải thích:** `GameState.MAP_SIZE = 7`; tọa độ hợp lệ từ 0 đến 6.

### 3. `0` và `1` trong map có nghĩa gì?

**Trả lời ngắn:** `0` là đường, `1` là vật cản.

### 4. Có bao nhiêu vật cản?

**Trả lời ngắn:** Tám vật cản, tạo từ bốn cặp đối xứng.

### 5. Phép đối xứng được tính thế nào?

**Trả lời ngắn:** `(row,col) → (6-row,6-col)`, hoặc với cell phẳng là `48-cell`.

### 6. Vì sao cần đối xứng?

**Trả lời ngắn:** Để hai phía có điều kiện bản đồ tương đương.

**Giải thích:** Vật cản, spawn và các cặp rương đối xứng 180 độ.

### 7. Vì sao đối xứng vẫn phải chạy BFS?

**Trả lời ngắn:** Đối xứng không bảo đảm mọi ô đường liên thông.

**Giải thích:** Các vật cản vẫn có thể tạo vùng bị cô lập. BFS loại các map đó.

### 8. BFS bắt đầu từ đâu?

**Trả lời ngắn:** Ô trung tâm `(3,3)`, cell 24.

### 9. BFS phải thăm bao nhiêu ô?

**Trả lời ngắn:** 41 ô đường.

**Giải thích:** 49 ô tổng trừ 8 vật cản.

### 10. Nếu không sinh được map hợp lệ thì sao?

**Trả lời ngắn:** Thử lại tối đa 1000 lần, sau đó ném lỗi.

**Giải thích:** Room lỗi không được thêm như một trận hoạt động hoàn chỉnh.

### 11. Map random có phải random matchmaking không?

**Trả lời ngắn:** Không.

**Giải thích:** Map ngẫu nhiên do `MapGenerator`; ghép người chơi dựa trên lời mời thuộc phần Toàn.

## B. Spawn và rương ban đầu

### 12. Hai Player xuất phát thế nào?

**Trả lời ngắn:** P1 ở một ô đường ngẫu nhiên hợp lệ, P2 ở ô đối xứng 180 độ.

### 13. Dữ liệu nào được reset khi bắt đầu trận?

**Trả lời ngắn:** Position, `matchScore` và `collectedTreasure`.

**Giải thích:** Rank, wins và thống kê tổng không thuộc điểm trận nên không reset ở `MapGenerator`.

### 14. Có bao nhiêu rương ban đầu?

**Trả lời ngắn:** Năm: hai NORMAL, hai SILVER và một GOLD.

### 15. Rương vàng nằm đâu?

**Trả lời ngắn:** Ở tâm `(3,3)`.

### 16. Điểm mỗi loại rương?

**Trả lời ngắn:** NORMAL 10, SILVER 20, GOLD 30.

## C. MOVE và va chạm

### 17. Client gửi tọa độ hay hướng?

**Trả lời ngắn:** Chỉ gửi hướng.

**Giải thích:** Message là `MOVE|UP/DOWN/LEFT/RIGHT`. Server tự tính ô đích từ Position hiện tại.

### 18. Tại sao không gửi tọa độ?

**Trả lời ngắn:** Để Client không tự quyết định vị trí hoặc dịch chuyển gian lận.

### 19. Server biết ai gửi MOVE bằng cách nào?

**Trả lời ngắn:** Từ `ClientHandler` của socket đang gửi.

**Giải thích:** `ClientHandler` truyền `this` vào `ServerManager.move()`, không lấy username từ payload MOVE.

### 20. Server kiểm tra gì trước khi di chuyển?

**Trả lời ngắn:** Phiên login hợp lệ, Player `PLAYING`, thuộc Room, Room `RUNNING`, hướng hợp lệ và ô đích hợp lệ.

### 21. Các va chạm được chặn là gì?

**Trả lời ngắn:** Ngoài biên, vật cản và vị trí đối thủ.

### 22. Dòng nào thực sự thay đổi vị trí?

**Trả lời ngắn:** `moving.setPosition(new Position(row, col))` trong `GameRoom.handleMove()`.

### 23. `handleMove()` trả `null` nghĩa là gì?

**Trả lời ngắn:** MOVE hợp lệ và Position đã được cập nhật.

### 24. Nếu direction là `3,4` thì sao?

**Trả lời ngắn:** Bị từ chối vì chỉ chấp nhận bốn chuỗi hướng.

### 25. Vì sao GamePanel cũng có `canMove()`?

**Trả lời ngắn:** Để khóa input và cải thiện UX, không thay thế validation ở Server.

## D. Rương và điểm

### 26. Khi nào gọi `collectTreasure()`?

**Trả lời ngắn:** Chỉ sau một MOVE hợp lệ.

### 27. Server phát hiện nhặt rương thế nào?

**Trả lời ngắn:** So sánh Position của Player với Position từng Treasure.

### 28. Điểm lấy từ field Client gửi hay từ type?

**Trả lời ngắn:** Server tính từ type bằng `Treasure.valueForType()`.

### 29. Sau khi nhặt rương, số lượng rương có giảm không?

**Trả lời ngắn:** Không; rương cũ được thay ngay bằng một rương mới.

### 30. Rương mới không được đặt ở đâu?

**Trả lời ngắn:** Không trên vật cản, hai Player hoặc rương khác.

### 31. Tại sao tạo danh sách ô trống thay vì random thử liên tục?

**Trả lời ngắn:** Để có giới hạn rõ ràng và không rơi vào vòng lặp vô hạn.

### 32. `TREASURE_COLLECTED` dùng làm gì?

**Trả lời ngắn:** Hiện thông báo và âm thanh.

### 33. HUD cập nhật điểm từ đâu?

**Trả lời ngắn:** Từ Player trong `GAME_STATE`, không tự cộng từ event.

### 34. `matchScore` và rank point có giống nhau không?

**Trả lời ngắn:** Không.

**Giải thích:** Sơn xử lý điểm rương trong trận 10/20/30. Tùng xử lý rank thắng/hòa/thua +3/+1/+0 sau kết quả.

## E. Đồng bộ và đồng thời

### 35. Hai Client nhận cùng state bằng cách nào?

**Trả lời ngắn:** Server encode một `GameState` rồi `sendToRoom()` cho cả hai.

### 36. Vì sao dùng full snapshot?

**Trả lời ngắn:** State nhỏ, Client đơn giản và dễ giữ đồng bộ.

### 37. `GAME_STATE` có những phần nào?

**Trả lời ngắn:** Room ID, map, hai Player, treasures, remaining time và status.

### 38. Hai người MOVE cùng lúc thì sao?

**Trả lời ngắn:** Chạy tuần tự dưới khóa Room.

### 39. Hai người cùng nhặt một rương thì sao?

**Trả lời ngắn:** Chỉ request chạy trước được tính.

**Giải thích:** Rương bị xóa/respawn trước khi request sau lấy khóa; ngoài ra Player đầu có thể chặn ô đích.

### 40. Tại sao broadcast cũng nằm trong khóa Room?

**Trả lời ngắn:** Để snapshot không bị một MOVE khác sửa xen giữa lúc encode/gửi.

### 41. `ConcurrentHashMap` có đủ để bảo vệ gameplay không?

**Trả lời ngắn:** Không.

**Giải thích:** Gameplay là chuỗi nhiều bước; cần khóa Room để biến cả chuỗi thành một transaction nhất quán.

### 42. Client tự gửi GAME_STATE thì sao?

**Trả lời ngắn:** `ClientHandler` trả `ERROR`; message này chỉ Server được gửi.

## F. Câu hỏi về source hiện tại

### 43. Vì sao `collectTreasure()` có thể trả `null` dù vừa chạm rương?

**Trả lời ngắn:** Source hiện tại có nhánh trap; trap tạo event riêng và trả `null` thay vì trả rương thưởng.

### 44. Stun có nằm trong bảng phân công gốc không?

**Trả lời ngắn:** Không; bảng gốc nói bản ZIP chưa có stun, nhưng repository hiện tại đã được mở rộng.

### 45. Nếu thầy hỏi code trap thì trả lời thế nào?

> Đây là phần mở rộng gameplay trong source hiện tại. Trap được giữ bí mật ở Server, không cộng số rương, có thể trừ điểm và tạo TRAP_TRIGGERED. Tuy nhiên phần em được phân công chính vẫn là map, MOVE, va chạm, rương thưởng và đồng bộ GAME_STATE.

## G. Khi không chắc

Không đoán. Trả lời:

> Theo source hiện tại, em xác nhận được dữ liệu vào ở method …, thay đổi state ở method … và message trả về là …. Phần ngoài phạm vi đó em xin mở đúng file để kiểm tra thay vì khẳng định sai.

