# 02. Sinh bản đồ, đối xứng và BFS

## 1. Điểm vào code

Mở theo thứ tự:

1. `server/GameRoom.java` → `startGame()`.
2. `server/MapGenerator.java` → `generate()`.
3. `server/MapGenerator.java` → `isConnected()`.
4. `client/BoardPanel.java` → `paintComponent()`.

`GameRoom.startGame()` gọi `MapGenerator.generate()` ở Server. Client không tự random map.

## 2. Hệ tọa độ

Bản đồ có kích thước `7 × 7`.

- `row`: hàng từ `0` đến `6`.
- `col`: cột từ `0` đến `6`.
- `map[row][col] == 0`: đường đi.
- `map[row][col] == 1`: vật cản.

Code cũng biểu diễn một ô bằng số phẳng:

```text
cell = row * 7 + col
row  = cell / 7
col  = cell % 7
```

Ví dụ:

| Tọa độ | Cell |
|---|---:|
| `(0,0)` | 0 |
| `(1,1)` | 8 |
| `(3,3)` | 24 |
| `(6,6)` | 48 |

## 3. Đối xứng 180 độ

Hàm:

```java
private int opposite(int cell) {
    return SIZE * SIZE - 1 - cell;
}
```

Với `SIZE = 7`, công thức là:

```text
opposite = 48 - cell
```

Tương đương theo tọa độ:

```text
(row, col) → (6-row, 6-col)
```

Ví dụ:

| Ô gốc | Ô đối xứng |
|---|---|
| `(0,0)` | `(6,6)` |
| `(1,1)` | `(5,5)` |
| `(2,4)` | `(4,2)` |
| `(3,3)` | `(3,3)` |

Tâm `(3,3)` đối xứng với chính nó nên được giữ làm ô đường và đặt rương vàng.

## 4. Tạo 8 vật cản

`generate()` tạo danh sách các cell ở nửa đầu bản đồ, xáo trộn rồi chọn bốn cell.

Với mỗi cell được chọn:

1. Đặt cell đó thành `1`.
2. Tính cell đối xứng.
3. Đặt cell đối xứng thành `1`.

Kết quả:

```text
4 cell gốc + 4 cell đối xứng = 8 vật cản
```

Tỷ lệ vật cản:

```text
8 / 49 ≈ 16,3%
```

## 5. Vì sao đối xứng chưa đủ?

Một map đối xứng vẫn có thể tạo vùng đường bị cô lập. Ví dụ, các tường đối xứng có thể bao kín một nhóm ô. Khi đó người chơi hoặc rương có thể nằm ở vùng không thể tiếp cận.

Vì vậy map phải vượt thêm kiểm tra liên thông bằng BFS.

## 6. BFS trong `isConnected()`

BFS là Breadth-First Search — duyệt theo chiều rộng bằng hàng đợi.

### Khởi tạo

```java
Queue<Integer> queue = new ArrayDeque<>();
queue.add(CENTER);
visited[3][3] = true;
```

Tâm luôn là đường nên có thể dùng làm điểm bắt đầu.

### Mỗi vòng lặp

1. Lấy một cell ở đầu queue.
2. Tăng biến `reachable`.
3. Xét bốn hàng xóm:
   - trên `(-1,0)`;
   - dưới `(1,0)`;
   - trái `(0,-1)`;
   - phải `(0,1)`.
4. Hàng xóm được thêm vào queue khi:
   - nằm trong map;
   - là đường `0`;
   - chưa được thăm.

### Điều kiện thành công

Map có 49 ô và đúng 8 vật cản, nên có 41 ô đường.

```java
return reachable == SIZE * SIZE - 8;
```

Nếu BFS từ tâm thăm được đủ 41 ô, mọi ô đường thuộc cùng một vùng và có đường đi nối với nhau.

## 7. Sinh vị trí xuất phát

Server duyệt các ô đường và giữ những ô có khoảng cách Manhattan đủ xa ô đối xứng.

```text
distance = |row1-row2| + |col1-col2|
```

Sau đó:

- chọn ngẫu nhiên một ô làm spawn P1;
- spawn P2 bằng `opposite(spawn1)`.

Hai spawn đối xứng giúp điều kiện bản đồ của hai người tương đương.

`resetPlayer()` đặt:

- vị trí mới;
- `matchScore = 0`;
- `collectedTreasure = 0`.

Nó không xóa rank hay wins vì đó là thống kê lâu dài.

## 8. Sinh năm rương ban đầu

Các vị trí rương cặp phải:

- nằm trên đường;
- không phải vật cản;
- cách mỗi spawn ít nhất hai bước Manhattan.

Server tạo:

```text
T1, T2: NORMAL, mỗi rương 10 điểm, đối xứng nhau
T3, T4: SILVER, mỗi rương 20 điểm, đối xứng nhau
T5: GOLD, 30 điểm, nằm ở tâm
```

Điều này giúp tổng khoảng cách tới từng loại rương của hai người cân bằng theo phép đối xứng.

## 9. Vì sao thử tối đa 1000 lần?

Random có thể tạo:

- map không liên thông;
- không còn vị trí spawn phù hợp;
- không đủ vị trí đặt cặp rương;
- map tái đấu giống map trước.

Thay vì vòng lặp vô hạn, code giới hạn 1000 lần. Nếu vẫn thất bại, Server ném `IllegalStateException` và không đưa Room lỗi vào hoạt động.

## 10. Client vẽ map

`BoardPanel.setSnapshot()` nhận `GameState` đã decode rồi gọi `repaint()`.

`paintComponent()` vẽ theo lớp:

1. khung bản đồ;
2. nền 49 ô;
3. vật cản nếu `map[row][col] == 1`;
4. các rương theo `state.getTreasures()`;
5. hai Player theo `Position`.

Client không chạy `MapGenerator` và không thay map.

## 11. Câu trả lời mẫu

### Tại sao map random vẫn bảo đảm chơi được?

> Sau khi đặt tám vật cản đối xứng, Server chạy BFS từ tâm. Map chỉ được chấp nhận khi BFS thăm đủ 41 ô đường, nghĩa là mọi vùng trống liên thông. Map không đạt sẽ bị bỏ và sinh lại, tối đa 1000 lần.

### Tại sao dùng đối xứng?

> Vật cản, spawn và các cặp rương được bố trí đối xứng 180 độ để hai phía có điều kiện tương đương. Đây là random map có kiểm soát, không phải random matchmaking.

### Client có thể tự sinh map khác không?

> Không. Chỉ Server chạy MapGenerator. Server encode map vào GAME_STATE và gửi cùng payload cho hai Client; Client chỉ decode và vẽ.

