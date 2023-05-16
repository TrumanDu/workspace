# python 学习笔记之入门

## 数据类型及流程结构

**常用数据类型：**

-   整型 `12`
-   浮点型 `123.456` `1.23456e2`
-   字符串型 `hello`
-   布尔型 `True` `False`
-   复数型 `3+5j`

**变量命名，PEP8 要求：**

-   用小写字母拼写，多个单词用下划线连接。
-   受保护的实例属性用单个下划线开头
-   私有的实例属性用两个下划线开头

**变量类型进行转换：**

-   int()：将一个数值或字符串转换成整数，可以指定进制。
-   float()：将一个字符串转换成浮点数。
-   str()：将指定的对象转换成字符串形式，可以指定编码。
-   chr()：将整数转换成该编码对应的字符串（一个字符）。
-   ord()：将字符串（一个字符）转换成对应的编码（整数）。

**运算符**

-   `[]` `[:]` 下标，切片
-   `**` 指数
-   `is` `is not` 身份运算符
-   `in` `not in` 成员运算符
-   `not` `or` `and` 逻辑运算符

**分支**

```python
if x > 1:
    y = 3 * x - 5
elif x >= -1:
    y = x + 2
else:
    y = 5 * x + 3
print('f(%.2f) = %.2f' % (x, y))
```

**循环**

```python
for x in range(1, 101):
    if x % 2 == 0:
        sum += x
print(sum)

while True:
    counter += 1
    number = int(input('请输入: '))
    if number < answer:
        print('大一点')
    elif number > answer:
        print('小一点')
    else:
        print('恭喜你猜对了!')
        break
print('你总共猜了%d次' % counter)
```

## 常用数据结构

### 字符串

定义

```python
a="3.1415926"
b = '1.123456'
s3 = '''
hello,
world!
'''
```

长度

```python
len(a)
```

成员运算

```python
s1 = 'hello, world'
print('wo' in s1)    # True
s2 = 'goodbye'
print(s2 in s1)      # False
```

索引和切片

```python
s = 'abc123456'
# 获取第一个字符
print(s[0], s[-N])    # a a
# i=2, j=5, k=1的正向切片操作
print(s[2:5])       # c12
```

编码/解码

```python
b = a.encode('utf-8')
c = a.encode('gbk')
print(b.decode('utf-8'))
print(c.decode('gbk'))
```

格式化字符串

```python
a = 3.1415926
print('{:.2f}'.format(a))# 保留小数点后两位
print('{:.2%}'.format(a))# 百分比格式
print('{:,}'.format(a))# 逗号分隔格式
print('{:.0f}'.format(a))# 不带小数
```

更多详见：[第 10 课：常用数据结构之字符串](https://github.com/jackfrued/Python-Core-50-Courses/blob/master/%E7%AC%AC10%E8%AF%BE%EF%BC%9A%E5%B8%B8%E7%94%A8%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B9%8B%E5%AD%97%E7%AC%A6%E4%B8%B2.md)

### 列表

定义

```python
items1 = [35, 12, 99, 68, 55, 87]
items2 = ['Python', 'Java', 'Go', 'Kotlin']
items1 = list(range(1, 10))
print(items1)    # [1, 2, 3, 4, 5, 6, 7, 8, 9]
items2 = list('hello')
print(items2)    # ['h', 'e', 'l', 'l', 'o']
```

列表运算

```python
items1 = [35, 12, 99, 68, 55, 87]
items2 = [45, 8, 29]

# 列表的拼接
items3 = items1 + items2
print(items3)    # [35, 12, 99, 68, 55, 87, 45, 8, 29]

# 列表的重复
items4 = ['hello'] * 3
print(items4)    # ['hello', 'hello', 'hello']

# 列表的成员运算
print('hello' in items4)    # True

# 获取列表的长度(元素个数)
size = len(items3)

# 列表的索引
print(items3[0], items3[-size])        # 35 35
items3[-1] = 100
print(items3[size - 1], items3[-1])    # 100 100

# 列表的切片
print(items3[:5])          # [35, 12, 99, 68, 55]
print(items3[4:])          # [55, 87, 45, 8, 100]
print(items3[-5:-7:-1])    # [55, 68]
print(items3[::-2])        # [100, 45, 55, 99, 35]

# 列表的比较运算
items5 = [1, 2, 3, 4]
items6 = list(range(1, 5))
# 两个列表比较相等性比的是对应索引位置上的元素是否相等
print(items5 == items6)    # True
```

列表遍历

```python
items = ['Python', 'Java', 'Go', 'Kotlin']

for index in range(len(items)):
    print(items[index])

for item in items:
    print(item)
```

列表操作

```python
# 添加
items.append('Swift')
items.insert(2, 'SQL')
# 删除
items.remove('Java')
items.pop(0)
# 清空
items.clear()
# 排序
items.sort()
# 反转
items.reverse()
```

更多详见 [第 08 课：常用数据结构之列表](https://github.com/jackfrued/Python-Core-50-Courses/blob/master/%E7%AC%AC08%E8%AF%BE%EF%BC%9A%E5%B8%B8%E7%94%A8%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B9%8B%E5%88%97%E8%A1%A8.md)

### 元祖

元组是不可变类型，通常不可变类型在创建时间和占用空间上面都优于对应的可变类型。

定义

```python
# 定义一个三元组
t1 = (30, 10, 55)
```

操作

```python
# 查看元组中元素的数量
print(len(t1), len(t2))
# 循环遍历元组中的元素
for member in t2:
    print(member)
# 成员运算
print(40 in t2)     # True
# 拼接
t3 = t1 + t2
# 切片
print(t3[::3])
```

元组和列表转换

```python
# 将元组转换成列表
info = (180, True, '四川成都')
print(list(info))
# 将列表转换成元组
fruits = ['apple', 'banana', 'orange']
print(tuple(fruits))
```

打包和解包

```python
# 打包
a = 1, 10, 100
print(type(a), a)    # <class 'tuple'> (1, 10, 100)
# 解包
i, j, k = a
print(i, j, k)       # 1 10 100
```

### 集合

定义

```python
set1 = {1, 2, 3}
set2 = set('hello')# {'h', 'l', 'o', 'e'}
set3 = set([1, 2, 3, 3, 2, 1])
```

操作

```python
# 创建一个空集合
set1 = set()
# 通过add方法添加元素
set1.add(33)
set1.update({1, 10, 100, 1000})
# 通过discard方法删除指定元素
set1.discard(100)
# 通过remove方法删除指定元素，建议先做成员运算再删除
# 否则元素如果不在集合中就会引发KeyError异常
if 10 in set1:
    set1.remove(10)
print(set1)    # {33, 1, 55, 1000}

# pop方法可以从集合中随机删除一个元素并返回该元素
print(set1.pop())

# clear方法可以清空整个集合
set1.clear()

set2 = {'Python', 'Java', 'Go', 'Swift'}
print('Ruby' in set2)    # False
```

### 字典

定义
```python
person = {
    'name': 'trumman, 'age': 55
}
person = dict(name='trumman', age=55)
```

操作
```python
# 查看多少组键值对
len(person)
# 检查name和tel两个键在不在person字典中
print('name' in person, 'tel' in person)    # True False
# 获取字典中所有的键
print(students.keys())      # dict_keys([1001, 1002, 1003])
# 获取字典中所有的值
print(students.values())    # dict_values([{...}, {...}, {...}])
# 获取字典中所有的键值对
print(students.items())     # dict_items([(1001, {...}), (1002, {....}), (1003, {...})])
# 对字典中所有的键值对进行循环遍历
for key, value in students.items():
    print(key, '--->', value)
```


## 函数和模块

可变参数
```python
# 用星号表达式来表示args可以接收0个或任意多个参数
def add(*args):
    total = 0
    # 可变参数可以放在for循环中取出每个参数的值
    for val in args:
        if type(val) in (int, float):
            total += val
    return total
```
模块
module1.py
```python
def foo():
    print('hello, world!')
```
```
import module1 as m1
from module1 import foo as f1

# 用“模块名.函数名”的方式（完全限定名）调用函数，
m1.foo()    # hello, world!
f1()    # hello, world!
```

### 装饰器
```python
import random
import time

from functools import wraps


def record_time(func):

    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f'{func.__name__}执行时间: {end - start:.3f}秒')
        return result

    return wrapper


@record_time
def download(filename):
    print(f'开始下载{filename}.')
    time.sleep(random.randint(2, 6))
    print(f'{filename}下载完成.')


@record_time
def upload(filename):
    print(f'开始上传{filename}.')
    time.sleep(random.randint(4, 8))
    print(f'{filename}上传完成.')


download('MySQL从删库到跑路.avi')
upload('Python从入门到住院.pdf')
# 取消装饰器
download.__wrapped__('MySQL必知必会.pdf')
upload = upload.__wrapped__
upload('Python从新手到大师.pdf')
```


## 面向对象

### 定义
```
class Student:
    """学生"""

    def __init__(self, name, age):
        """初始化方法"""
        self.name = name
        self.__age = age

    def study(self, course_name):
        """学习"""
        print(f'{self.name}正在学习{course_name}.')

    def play(self):
        """玩耍"""
        print(f'{self.name}正在玩游戏.')

    def __repr__(self):
        """类似于java中toString方法，将对象内容按如下输出"""
        return f'{self.name}: {self.age}'


stu1 = Student('truman', 18)

```
__age表示一个私有属性


### 进阶
```python
class Student:

    def __init__(self, name, age):
        self.__name = name
        self.__age = age

    # 属性访问器(getter方法) - 获取__name属性
    @property
    def name(self):
        return self.__name
    
    # 属性修改器(setter方法) - 修改__name属性
    @name.setter
    def name(self, name):
        # 如果name参数不为空就赋值给对象的__name属性
        # 否则将__name属性赋值为'无名氏'，有两种写法
        # self.__name = name if name else '无名氏'
        self.__name = name or '无名氏'
    
    @property
    def age(self):
        return self.__age


stu = Student('王大锤', 20)
print(stu.name, stu.age)    # 王大锤 20
```
通过一个@符号表示将装饰器应用于类、函数或方法

### 静态方法和类方法
```python
    @staticmethod
    def is_valid(a, b, c):
        """判断三条边长能否构成三角形(静态方法)"""
        return a + b > c and b + c > a and a + c > b

    @classmethod
    def is_valid(cls, a, b, c):
        """判断三条边长能否构成三角形(类方法)"""
        return a + b > c and b + c > a and a + c > b
```

### 继承与多态
```python
class Person:
    """人类"""

    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def eat(self):
        print(f'{self.name}正在吃饭.')
    
    def sleep(self):
        print(f'{self.name}正在睡觉.')


class Student(Person):
    """学生类"""
    
    def __init__(self, name, age):
        # super(Student, self).__init__(name, age)
        super().__init__(name, age)
    
    def study(self, course_name):
        print(f'{self.name}正在学习{course_name}.')


class Teacher(Person):
    """老师类"""

    def __init__(self, name, age, title):
        # super(Teacher, self).__init__(name, age)
        super().__init__(name, age)
        self.title = title
    
    def teach(self, course_name):
        print(f'{self.name}{self.title}正在讲授{course_name}.')
```

### 枚举类
```pyton
from enum import Enum


class Suite(Enum):
    """花色(枚举)"""
    SPADE, HEART, CLUB, DIAMOND = range(4)
```

## 线程

方式一：通过继承 Thread 类创建线程
```python
import threading

# 自定义线程类
class MyThread(threading.Thread):
    def run(self):
        print("Hello from thread")

# 创建线程实例并启动线程
thread = MyThread()
thread.start()
```
方式二：通过传递函数创建线程

```python
import threading

# 线程执行的函数
def my_function():
    print("Hello from thread")

# 创建线程实例并启动线程
thread = threading.Thread(target=my_function)
thread.start()

```
方式三：使用线程池
```python
import concurrent.futures

# 线程执行的函数
def my_function():
    print("Hello from thread")

# 创建线程池
with concurrent.futures.ThreadPoolExecutor() as executor:
    # 提交任务给线程池
    future = executor.submit(my_function)
    # 获取任务的结果
    result = future.result()

```


## 文件与异常
### 读写文件
```python
file = open('致橡树.txt', 'r', encoding='utf-8')
for line in file:
    print(line, end='')
file.close()

file = open('致橡树.txt', 'r', encoding='utf-8')
lines = file.readlines()
for line in lines:
    print(line, end='')
file.close()

file = open('致橡树.txt', 'a', encoding='utf-8')
file.write('\n标题：《致橡树》')
file.write('\n作者：舒婷')
file.write('\n时间：1977年3月')
file.close()
```
大文件读取
```python
try:
    with open('guido.jpg', 'rb') as file1, open('吉多.jpg', 'wb') as file2:
        data = file1.read(512)
        while data:
            file2.write(data)
            data = file1.read()
except FileNotFoundError:
    print('指定的文件无法打开.')
except IOError:
    print('读写文件时出现错误.')
print('程序执行结束.')
```


### 异常

python中所有的异常都是`BaseException`的子类型，它有四个直接的子类，分别是：`SystemExit`、`KeyboardInterrupt`、`GeneratorExit`和`Exception`。其中，SystemExit表示解释器请求退出，KeyboardInterrupt是用户中断程序执行（按下Ctrl+c），GeneratorExit表示生成器发生异常通知退出。

```python
# 定义自定义异常类
class MyException(Exception):
    pass

# 抛出自定义异常
raise MyException("This is a custom exception")

# 定义自定义异常类
class MyException(Exception):
    def __init__(self, message, code):
        super().__init__(message)
        self.code = code

    def get_code(self):
        return self.code

# 抛出自定义异常
raise MyException("This is a custom exception", 1001)

```


## 参考

1.[Python-100-Days](https://github.com/jackfrued/Python-100-Days) 
2.[Python-Core-50-Courses](https://github.com/jackfrued/Python-Core-50-Courses)
