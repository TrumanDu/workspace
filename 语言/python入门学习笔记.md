# python 入门笔记

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


## 面向对象

## 进程和线程

## 文件与异常


## 参考

1.[Python-100-Days](https://github.com/jackfrued/Python-100-Days) 
2.[Python-Core-50-Courses](https://github.com/jackfrued/Python-Core-50-Courses)
