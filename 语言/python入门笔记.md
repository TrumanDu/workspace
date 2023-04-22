# python 入门笔记

## 基础

### 数据类型及流程结构

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
