# Rock Paper Scissors - Writeup

## 1. Tổng quan bài

Chương trình là một game **Rock, Paper, Scissors** đơn giản.

Luật của bài:

- Người chơi chọn `rock`, `paper`, hoặc `scissors`.
- Máy tính chọn ngẫu nhiên một trong ba lựa chọn.
- Nếu người chơi thắng **5 lần liên tiếp**, chương trình sẽ in ra flag.

Đoạn kiểm tra số lần thắng nằm trong `main()`:

```c
if (play()) {
  wins++;
} else {
  wins = 0;
}

if (wins >= 5) {
  puts("Congrats, here's the flag!");
  puts(flag);
}
```

Biến `wins` dùng để đếm số lần thắng liên tiếp. Nếu thua một ván, `wins` bị reset về `0`.

---

## 2. Phân tích hàm `play()`

Hàm `play()` xử lý một lượt chơi:

```c
bool play () {
  char player_turn[100];
  srand(time(0));
  int r;

  printf("Please make your selection (rock/paper/scissors):\n");
  r = tgetinput(player_turn, 100);

  int computer_turn = rand() % 3;
  printf("You played: %s\n", player_turn);
  printf("The computer played: %s\n", hands[computer_turn]);

  if (strstr(player_turn, loses[computer_turn])) {
    puts("You win! Play again?");
    return true;
  } else {
    puts("Seems like you didn't win this time. Play again?");
    return false;
  }
}
```

Trong đó:

```c
char player_turn[100];
```

`player_turn` là mảng ký tự dùng để chứa lựa chọn mà người chơi nhập vào.

Ví dụ:

- Nhập `rock` thì `player_turn = "rock"`
- Nhập `paper` thì `player_turn = "paper"`
- Nhập `rockpaperscissors` thì `player_turn = "rockpaperscissors"`

---

## 3. Mảng lựa chọn của game

Chương trình có hai mảng quan trọng:

```c
char* hands[3] = {"rock", "paper", "scissors"};
char* loses[3] = {"paper", "scissors", "rock"};
```

Mảng `hands` chứa lựa chọn của máy:

| Giá trị `computer_turn` | Máy chọn |
|---|---|
| 0 | rock |
| 1 | paper |
| 2 | scissors |

Mảng `loses` chứa lựa chọn có thể thắng máy:

| Máy chọn | Người chơi cần chọn để thắng |
|---|---|
| rock | paper |
| paper | scissors |
| scissors | rock |

Ví dụ nếu máy chọn `rock`, người chơi cần chọn `paper` để thắng.

---

## 4. Lỗ hổng của chương trình

Lỗi nằm ở đoạn kiểm tra thắng:

```c
if (strstr(player_turn, loses[computer_turn])) {
```

Hàm `strstr(a, b)` kiểm tra chuỗi `b` có nằm bên trong chuỗi `a` hay không.

Ví dụ:

```c
strstr("hello paper test", "paper")
```

Kết quả là đúng vì chuỗi `"paper"` xuất hiện bên trong `"hello paper test"`.

Vấn đề là chương trình không kiểm tra người chơi nhập chính xác `rock`, `paper`, hoặc `scissors`.

Đáng lẽ chương trình nên dùng `strcmp()` để so sánh chính xác:

```c
strcmp(player_turn, loses[computer_turn]) == 0
```

Nhưng chương trình lại dùng `strstr()`, nên chỉ cần input của người chơi **có chứa** lựa chọn thắng máy là được tính thắng.

---

## 5. Ý tưởng khai thác

Máy có thể chọn một trong ba lựa chọn:

```text
rock
paper
scissors
```

Để thắng từng trường hợp, người chơi cần có:

```text
paper
scissors
rock
```

Vì chương trình chỉ kiểm tra chuỗi con bằng `strstr()`, ta có thể nhập một chuỗi chứa đủ cả ba từ:

```text
rockpaperscissors
```

Chuỗi này chứa:

- `rock`
- `paper`
- `scissors`

Vậy dù máy chọn gì, điều kiện sau vẫn đúng:

```c
strstr(player_turn, loses[computer_turn])
```

Cụ thể:

### Trường hợp 1: Máy chọn rock

```c
loses[0] = "paper";
strstr("rockpaperscissors", "paper");
```

Có `paper` trong input, nên người chơi thắng.

### Trường hợp 2: Máy chọn paper

```c
loses[1] = "scissors";
strstr("rockpaperscissors", "scissors");
```

Có `scissors` trong input, nên người chơi thắng.

### Trường hợp 3: Máy chọn scissors

```c
loses[2] = "rock";
strstr("rockpaperscissors", "rock");
```

Có `rock` trong input, nên người chơi thắng.

Như vậy input `rockpaperscissors` sẽ luôn thắng.

---

## 6. Khai thác thủ công

Mỗi lượt chơi ta nhập:

```text
1
rockpaperscissors
```

Trong đó:

- `1` là chọn chơi game.
- `rockpaperscissors` là input giúp thắng chắc chắn.

Cần thắng 5 lần liên tiếp, nên nhập 5 lần:

```text
1
rockpaperscissors
1
rockpaperscissors
1
rockpaperscissors
1
rockpaperscissors
1
rockpaperscissors
```

Sau 5 lần thắng, chương trình sẽ in flag.

---

## 7. Payload chạy local

Có thể tự động gửi input bằng Python:

```bash
python3 -c 'print(("1\nrockpaperscissors\n")*5)' | ./vuln
```

Nếu binary có tên khác thì thay `./vuln` bằng tên file thực thi tương ứng.

---

## 8. Payload chạy remote

Nếu đề bài cung cấp host và port, dùng:

```bash
python3 -c 'print(("1\nrockpaperscissors\n")*5)' | nc HOST PORT
```

Thay:

- `HOST` bằng địa chỉ server.
- `PORT` bằng port của challenge.

Ví dụ dạng chung:

```bash
python3 -c 'print(("1\nrockpaperscissors\n")*5)' | nc challenge.server.com 12345
```

---

## 9. Nguyên nhân lỗi

Nguyên nhân chính là chương trình dùng sai hàm kiểm tra chuỗi.

Đoạn code lỗi:

```c
if (strstr(player_turn, loses[computer_turn])) {
```

`strstr()` chỉ kiểm tra chuỗi con, không kiểm tra bằng tuyệt đối.

Do đó input như sau vẫn được chấp nhận:

```text
rockpaperscissors
```

Mặc dù đây không phải một lựa chọn hợp lệ trong game.

Cách sửa đúng hơn là dùng `strcmp()`:

```c
if (strcmp(player_turn, loses[computer_turn]) == 0) {
```

Khi đó người chơi bắt buộc phải nhập chính xác lựa chọn thắng máy.

---

## 10. Kết luận

Bài này là một lỗi **logic bug** trong xử lý input.

Không cần khai thác buffer overflow, format string hay đoán random. Chỉ cần hiểu cách `strstr()` hoạt động là có thể bypass luật chơi.

Payload chính:

```text
rockpaperscissors
```

Do input này chứa đủ cả `rock`, `paper`, và `scissors`, nên dù máy chọn gì thì điều kiện thắng vẫn đúng. Lặp lại 5 lần sẽ lấy được flag.
