# 06. Đồng thời và Server authoritative

## 1. Server authoritative là gì?

Server authoritative nghĩa là Server giữ trạng thái chuẩn và đưa ra quyết định cuối cùng.

Client được phép gửi:

```text
“Tôi muốn đi UP”
```

Client không được quyền khẳng định:

```text
“Tôi đang ở (2,3)”
“Tôi vừa được cộng 30 điểm”
“Rương mới nằm ở (4,5)”
```

## 2. Vì sao cần thiết?

Nếu Client tự quyết định:

- người chơi có thể sửa chương trình để đi xuyên tường;
- hai Client có thể random khác nhau;
- hai người cùng nhặt một rương có thể cùng được điểm;
- mạng trễ có thể làm thứ tự state khác nhau;
- điểm trên hai màn hình có thể lệch.

## 3. Các lớp validation

### Client-side

`GamePanel.canMove()` khóa nút khi Client biết mình không thể thao tác.

Mục đích: trải nghiệm người dùng.

### ServerManager

Kiểm tra:

- handler đúng phiên login;
- Player tồn tại;
- Player `PLAYING`;
- Player thuộc một Room active.

### GameRoom

Kiểm tra:

- Room đang `RUNNING`;
- direction hợp lệ;
- không ngoài biên;
- không vào vật cản;
- không đè lên đối thủ.

Server-side mới là lớp có thẩm quyền.

## 4. Nguồn đồng thời

Server sử dụng một `ClientHandler` cho mỗi kết nối. Hai Client có thể gửi MOVE gần như cùng lúc từ hai thread khác nhau.

Timer cũng có thread riêng, nhưng phần timer thuộc Tùng. Sơn cần hiểu nó là một nguồn cạnh tranh có thể cùng truy cập Room.

## 5. Khóa ServerManager và Room

`ServerManager.move()` là `synchronized`, sau đó khóa đúng Room:

```java
public synchronized void move(...) {
    ...
    synchronized (room) {
        // di chuyển
        // nhặt rương
        // respawn
        // gửi event
        // gửi GAME_STATE
    }
}
```

Phạm vi khóa Room bao trọn toàn bộ transaction gameplay.

## 6. Tình huống hai người cùng tranh một rương

Giả sử rương ở `(3,3)`.

1. Alex gửi MOVE từ bên trái.
2. Nina gửi MOVE từ bên phải.
3. Thread Alex lấy khóa Room trước.
4. Alex vào `(3,3)`, rương bị xóa, điểm được cộng, rương mới được sinh.
5. Server gửi state của Alex.
6. Thread Alex nhả khóa.
7. Thread Nina lấy khóa.
8. Nina không thể vào `(3,3)` vì Alex đang đứng ở đó.

Chỉ Alex được tính rương.

## 7. Tình huống hai người cùng đi vào một ô trống

Request đầu chiếm ô. Request sau kiểm tra vị trí đối thủ mới nhất và bị từ chối.

Không tồn tại snapshot trung gian có hai Player chồng nhau.

## 8. Tại sao broadcast nằm trong khóa?

Nếu nhả khóa trước khi encode/send:

1. MOVE A cập nhật state.
2. MOVE B cập nhật tiếp.
3. Thread A mới encode.

Khi đó A có thể vô tình gửi state của B hoặc thứ tự khác nhau giữa hai Client.

Hiện tại state được encode và gửi khi Room vẫn đang khóa, nên mỗi transaction có một snapshot rõ ràng.

## 9. Các hành vi gian lận bị chặn

| Hành vi | Nơi chặn |
|---|---|
| Gửi tọa độ thay hướng | `GameRoom.handleMove()` chỉ chấp nhận bốn direction |
| Gửi MOVE khi chưa login | `ServerManager.move()` |
| Gửi MOVE khi ở Lobby | kiểm tra `PLAYING` |
| Điều khiển Room khác | tìm Room theo object Player của handler |
| Tự gửi GAME_STATE | `ClientHandler` từ chối message chỉ dành cho Server |
| Tự gửi TREASURE_COLLECTED | `ClientHandler` từ chối |
| Tự cộng điểm ở UI | snapshot tiếp theo từ Server ghi đè giao diện |

## 10. `ConcurrentHashMap` có thay thế khóa Room không?

Không.

`ConcurrentHashMap` bảo vệ cấu trúc map cho từng thao tác đơn lẻ. Nó không biến chuỗi thao tác sau thành một transaction:

```text
kiểm tra vị trí
→ đổi Position
→ tìm rương
→ cộng điểm
→ respawn
→ encode
→ gửi hai Client
```

Khóa Room mới giữ toàn bộ chuỗi đó nhất quán.

## 11. Câu trả lời mẫu

### Hai Client gửi cùng lúc thì Server xử lý thế nào?

> Hai handler có thể chạy đồng thời, nhưng Server khóa đúng GameRoom. Mỗi MOVE hoàn tất toàn bộ bước di chuyển, nhặt rương, respawn và broadcast trước khi MOVE còn lại chạy, nên state không bị xen kẽ.

### Vì sao không tin kiểm tra ở GamePanel?

> GamePanel chỉ hỗ trợ UX và có thể chứa snapshot cũ hoặc bị sửa. Server mới giữ state mới nhất và phải kiểm tra lại toàn bộ luật.

