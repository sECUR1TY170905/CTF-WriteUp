# PicoCTF - format string 0

## Thông tin bài tập
- **Tên bài**: format string 0
- **Dạng bài**: Binary Exploitation / Pwn
- **Lỗ hổng**: Format String 

## Phân tích mã nguồn

Bài tập cung cấp cho chúng ta một chương trình C và yêu cầu phục vụ các khách hàng bằng cách chọn hamburger từ menu.

Mã nguồn được cho như sau:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>

#define BUFSIZE 32
#define FLAGSIZE 64

char flag[FLAGSIZE];

void sigsegv_handler(int sig) {
    printf("\n%s\n", flag);
    fflush(stdout);
    exit(1);
}

int on_menu(char *burger, char *menu[], int count) {
    for (int i = 0; i < count; i++) {
        if (strcmp(burger, menu[i]) == 0)
            return 1;
    }
    return 0;
}

void serve_patrick();

void serve_bob();


int main(int argc, char **argv){
    FILE *f = fopen("flag.txt", "r");
    if (f == NULL) {
        printf("%s %s", "Please create 'flag.txt' in this directory with your",
                        "own debugging flag.\n");
        exit(0);
    }

    fgets(flag, FLAGSIZE, f);
    signal(SIGSEGV, sigsegv_handler);

    gid_t gid = getegid();
    setresgid(gid, gid, gid);

    serve_patrick();
  
    return 0;
}

void serve_patrick() {
    printf("%s %s\n%s\n%s %s\n%s",
            "Welcome to our newly-opened burger place Pico 'n Patty!",
            "Can you help the picky customers find their favorite burger?",
            "Here comes the first customer Patrick who wants a giant bite.",
            "Please choose from the following burgers:",
            "Breakf@st_Burger, Gr%114d_Cheese, Bac0n_D3luxe",
            "Enter your recommendation: ");
    fflush(stdout);

    char choice1[BUFSIZE];
    scanf("%s", choice1);
    char *menu1[3] = {"Breakf@st_Burger", "Gr%114d_Cheese", "Bac0n_D3luxe"};
    if (!on_menu(choice1, menu1, 3)) {
        printf("%s", "There is no such burger yet!\n");
        fflush(stdout);
    } else {
        int count = printf(choice1);
        if (count > 2 * BUFSIZE) {
            serve_bob();
        } else {
            printf("%s\n%s\n",
                    "Patrick is still hungry!",
                    "Try to serve him something of larger size!");
            fflush(stdout);
        }
    }
}

void serve_bob() {
    printf("\n%s %s\n%s %s\n%s %s\n%s",
            "Good job! Patrick is happy!",
            "Now can you serve the second customer?",
            "Sponge Bob wants something outrageous that would break the shop",
            "(better be served quick before the shop owner kicks you out!)",
            "Please choose from the following burgers:",
            "Pe%to_Portobello, $outhwest_Burger, Cla%sic_Che%s%steak",
            "Enter your recommendation: ");
    fflush(stdout);

    char choice2[BUFSIZE];
    scanf("%s", choice2);
    char *menu2[3] = {"Pe%to_Portobello", "$outhwest_Burger", "Cla%sic_Che%s%steak"};
    if (!on_menu(choice2, menu2, 3)) {
        printf("%s", "There is no such burger yet!\n");
        fflush(stdout);
    } else {
        printf(choice2);
        fflush(stdout);
    }
}
```

## Cách giải

Chúng ta cần chú ý đến hàm `main`, chương trình đọc cờ vào biến `flag` và thiết lập một signal handler cho `SIGSEGV` (Segmentation Fault):
```c
    signal(SIGSEGV, sigsegv_handler);
```
Hàm `sigsegv_handler` sẽ in ra flag nếu chương trình gặp lỗi Segmentation Fault:
```c
void sigsegv_handler(int sig) {
    printf("\n%s\n", flag);
    fflush(stdout);
    exit(1);
}
```
Vậy mục tiêu của chúng ta là **gây ra lỗi Segmentation Fault (SIGSEGV)** để chương trình tự động in cờ.

### Bước 1: Vượt qua `serve_patrick()`
Hàm này yêu cầu ta nhập 1 trong 3 loại burger: `Breakf@st_Burger`, `Gr%114d_Cheese`, `Bac0n_D3luxe`.
Sau đó nó dùng `printf(choice1)` (đây là lỗ hổng format string do gọi printf trực tiếp với input của người dùng) và kiểm tra:
```c
        int count = printf(choice1);
        if (count > 2 * BUFSIZE) {
            serve_bob();
        }
```
Ở đây `BUFSIZE` = 32, do đó chúng ta cần `count > 64`. Lựa chọn `Gr%114d_Cheese` là phù hợp vì `%114d` sẽ yêu cầu `printf` in ra số nguyên ở định dạng có độ rộng 114 ký tự. Khi đó độ dài tổng cộng chuỗi in ra sẽ lớn hơn 64, giúp ta vượt qua kiểm tra và tiến tới `serve_bob()`.

**Input 1:** `Gr%114d_Cheese`

### Bước 2: Vượt qua `serve_bob()` và kích hoạt SIGSEGV
Hàm này tiếp tục yêu cầu chọn burger từ menu: `Pe%to_Portobello`, `$outhwest_Burger`, `Cla%sic_Che%s%steak`.
Nó cũng bị lỗi format string ở đoạn `printf(choice2);`.
Để tạo ra lỗi SIGSEGV, ta cần buộc chương trình truy cập vào một vùng nhớ không hợp lệ. Chuỗi `Cla%sic_Che%s%steak` có chứa `%s`, format specifier này yêu cầu `printf` đọc một chuỗi từ địa chỉ con trỏ trên stack. Do chúng ta không kiểm soát stack một cách cẩn thận ở đây, `%s` sẽ đọc nhầm một địa chỉ rác trên stack, rất có khả năng địa chỉ đó không hợp lệ, dẫn đến lỗi Segmentation Fault.

**Input 2:** `Cla%sic_Che%s%steak`

## Kết quả
Sau khi nhập 2 chuỗi trên lần lượt, chương trình sẽ gặp lỗi bộ nhớ và kích hoạt `sigsegv_handler`, từ đó in ra cờ (flag).
