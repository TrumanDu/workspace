# python 入门笔记

## 基础

### 数据类型及流程结构

**常用数据类型：**
- 整型 `12`
- 浮点型 `123.456` `1.23456e2`
- 字符串型 `hello`
- 布尔型 `True` `False`
- 复数型 `3+5j`

**变量命名，PEP8要求：**
- 用小写字母拼写，多个单词用下划线连接。
- 受保护的实例属性用单个下划线开头
- 私有的实例属性用两个下划线开头

**变量类型进行转换：**
- int()：将一个数值或字符串转换成整数，可以指定进制。
- float()：将一个字符串转换成浮点数。
- str()：将指定的对象转换成字符串形式，可以指定编码。
- chr()：将整数转换成该编码对应的字符串（一个字符）。
- ord()：将字符串（一个字符）转换成对应的编码（整数）。

**运算符**
- `[]` `[:]` 下标，切片
- `**` 指数
- `is` `is not` 身份运算符
- `in` `not in` 成员运算符
- `not` `or` `and` 逻辑运算符

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

### 函数和模块

### 字符串与常用数据结构

## 进阶

### 面向对象

### 进程和线程

### 文件与异常

## 工程化

### 依赖管理

1. `pip freeze > requirements.txt`
2. `pip install pipreqs`
   `pipreqs ./ --encoding=utf8 --force`

### log 记录

简单使用

```python
import logging

logging.basicConfig(format='%(levelname)s - %(asctime)s - %(message)s',level=logging.DEBUG)
logging.warning('this is %s log','warning')
logging.info('info log')
logging.error('error log')
```

运行代码输出：

```
WARNING - 2023-04-21 10:27:38,631 - this is warning log
INFO - 2023-04-21 10:27:38,631 - info log
ERROR - 2023-04-21 10:27:38,631 - error log

```

#### yaml 配置 log

采用 YAML 格式，用于新的基于字典的方法

```yaml
version: 1
formatters:
  simple:
    format: '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
handlers:
  console:
    class: logging.StreamHandler
    level: DEBUG
    formatter: simple
    stream: ext://sys.stdout
loggers:
  app:
    level: DEBUG
    handlers: [ console ]
    propagate: no
root:
  level: DEBUG
  handlers: [ console ]
```

使用如下：

```python
import logging.config

import yaml

with open(file='conf/logging.yaml', mode='r', encoding='utf8') as file:
    logging_yaml = yaml.load(stream=file, Loader=yaml.FullLoader)
    logging.config.dictConfig(config=logging_yaml)

logger = logging.getLogger('app')
if __name__ == '__main__':
    logger.warning('this is %s log', 'warning')
    logger.info('info log')
    logger.error('error log')
```

### 配置文件

#### 环境变量中配置信息

```python
import os

secret_key = os.environ.get('SECRET_KEY', 'jajajajhh')
```

#### yaml 配置文件

```python
class Configure:
    def __init__(self):
        self.map_parameters = {}

        self.cfg_file = os.path.abspath(os.path.join(os.path.dirname(__file__), ".conf/conf.yaml"))
        self.init_local()

    def init_local(self):
        if os.path.exists(self.cfg_file):
            with open(self.cfg_file, "r+", encoding="utf-8") as reader:
                configure = yaml.load(reader, Loader=yaml.SafeLoader)
                if "hbase" not in configure.keys():
                    raise ValueError("not  hbase tables configure")
                self.map_parameters["hbase"] = configure["hbase"]
                self.map_parameters["hbase.common_param"] = configure["hbase"]["common_param"]
                self.map_parameters["hbase.tables"] = configure["hbase"]["tables"]
```

使用如下：

```python
conf = Configure()
hbase_param = conf.map_parameters["hbase"]
```

## 可视化

## 参考

1.[Python-100-Days](https://github.com/jackfrued/Python-100-Days)
