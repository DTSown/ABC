# 04. Nhặt rương, cộng điểm và sinh rương mới

## 1. Điểm vào code

- `model/Treasure.java`: `valueForType()`.
- `server/GameRoom.java`: `collectTreasure()`.
- `server/GameRoom.java`: `createReplacementTreasure()`.
- `server/ServerManager.java`: `move()`.
- `client/GamePanel.java`: `showTreasureCollected()`, `setGameState()`.

## 2. Khi nào kiểm tra nhặt rương?

Chỉ sau một MOVE hợp lệ:

```java
String error = room.handleMove(...);
if (error == null) {
    Treasure collected = room.collectTreasure(player);
}
```

Nếu bước đi bị tường/biên/đối thủ chặn, Server không kiểm tra nhặt rương.

## 3. Tìm rương tại vị trí Player

`collectTreasure()` duyệt danh sách rương và so sánh:

```text
player.row == treasure.row
player.col == treasure.col
```

Nếu không có rương trùng vị trí, trả `null` và điểm không đổi.

## 4. Quy tắc điểm

`Treasure.valueForType()` là nguồn ánh xạ duy nhất:

| Type | Điểm |
|---|---:|
| `NORMAL` | 10 |
| `SILVER` | 20 |
| `GOLD` | 30 |

Server tính lại điểm từ type. Client không gửi số điểm muốn được cộng.

Với rương thưởng:

```java
player.setMatchScore(player.getMatchScore() + value);
player.setCollectedTreasure(player.getCollectedTreasure() + 1);
```

`matchScore` là điểm trong trận, khác với điểm rank sau kết quả.

## 5. Vì sao tạo rương thay thế trước khi xóa rương cũ?

Code tạo `replacement` trước. Nếu không thể tìm ô hợp lệ, method ném lỗi trước khi thay đổi danh sách hiện tại.

Sau khi tạo thành công:

1. xóa rương đã nhặt;
2. xóa nó khỏi tập bẫy nếu cần;
3. thêm rương thay thế;
4. xử lý điểm hoặc event bẫy.

Nhờ vậy state không rơi vào tình trạng mất rương giữa chừng.

## 6. Chọn ô respawn

`createReplacementTreasure()` tạo mảng `occupied[7][7]`.

Đánh dấu occupied cho:

- vị trí Player 1;
- vị trí Player 2;
- tất cả rương còn lại.

Sau đó duyệt 49 ô và chỉ thêm vào `available` nếu:

```text
map[row][col] == 0
và occupied[row][col] == false
```

Vật cản bị loại bằng điều kiện map. Player và rương bị loại bằng `occupied`.

Server chọn ngẫu nhiên:

- một type trong `NORMAL`, `SILVER`, `GOLD`;
- một Position trong `available`;
- một ID mới dạng UUID.

## 7. Tại sao không random ô rồi thử lại vô hạn?

Nếu liên tục random một ô bất kỳ, có thể lặp nhiều lần vào vật cản, Player hoặc rương. Trong trường hợp không còn ô phù hợp, vòng lặp có thể không dừng.

Code hiện tại duyệt hữu hạn 49 ô, tạo danh sách hợp lệ rồi chọn một phần tử. Cách này có điểm dừng rõ ràng.

## 8. Thứ tự message

Với rương thưởng, Server gửi:

```text
TREASURE_COLLECTED|username;type;value;totalScore
GAME_STATE|...
```

### `TREASURE_COLLECTED`

Dùng cho:

- dòng thông báo;
- hiệu ứng âm thanh;
- cho cả hai biết ai vừa nhặt rương.

Client không dùng event này để tự cộng HUD.

### `GAME_STATE`

Chứa:

- điểm mới;
- số rương đã nhặt;
- danh sách rương sau respawn;
- vị trí hai người.

Đây mới là nguồn dữ liệu để cập nhật HUD và Board.

## 9. Hai người tranh cùng một rương

Giả sử cả hai cùng muốn đi vào ô rương.

1. Khóa Room cho một request chạy trước.
2. Người đầu MOVE vào ô, nhặt rương, rương cũ bị xóa và rương mới được sinh.
3. Request thứ hai chạy sau.
4. Nếu người đầu vẫn đứng trên ô đó, request thứ hai bị chặn do va chạm Player.
5. Dù trường hợp vị trí thay đổi, rương cũ không còn trong danh sách nên không thể cộng lần hai.

Kết quả: một rương chỉ được tính một lần.

## 10. Lưu ý về bẫy trong source hiện tại

Repository hiện tại có phần mở rộng rương bẫy:

- trap không cộng điểm và count;
- có thể trừ tối đa 10 điểm;
- tạo `TRAP_TRIGGERED`;
- `collectTreasure()` trả `null` ở nhánh trap.

Do đó `null` có thể nghĩa là:

1. không có rương tại vị trí; hoặc
2. vừa xử lý bẫy, event nằm trong `pendingTrapEvents`.

Khi trình bày đúng phần được phân công, hãy giải thích luồng rương thưởng trước. Nếu thầy chỉ vào nhánh `trap`, nói đây là phần mở rộng hiện có của gameplay.

## 11. Câu trả lời mẫu

### Điểm được cộng ở Client hay Server?

> Server. Sau MOVE hợp lệ, GameRoom tìm rương tại vị trí Player, tra điểm bằng type rồi cập nhật matchScore. Client chỉ hiển thị state được Server gửi về.

### Vì sao rương luôn xuất hiện lại hợp lệ?

> Server duyệt các ô đường, loại vị trí hai Player và các rương còn lại, rồi chọn ngẫu nhiên trong danh sách ô hợp lệ. Vì vậy rương mới không nằm trên tường, Player hoặc rương khác.

