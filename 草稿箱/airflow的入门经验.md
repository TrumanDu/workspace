# airflow 的入门经验

## 安装 airflow

### 升级 docker 与 docker-compose 版本

`Install Docker Compose v2.14.0 or newer on your workstation.`

这里记录一下我在 ubuntu 里升级方法。
首先升级相应的 docker 版本

```
apt-get update
apt upgrade docker-ce
```

由于 ubuntu 镜像源较旧，无法更新到最新版 docker-compose，接下来手动升级 docker-compose。
在手动升级之前，要确保 docker-ce 支持的最高 docker-compose[版本](https://docs.docker.com/compose/compose-file/compose-versioning/)。

![Img](http://image.trumandu.top/yank-note-picgo-img-20240413105720.png)

手动在[https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)下载 docker-compose 安装包。

首先找到 docker-compose 地址

```
root@lab:/# which docker-compose
/usr/bin/docker-compose
```

用下载的文件替换如上文件即可。

### 构建本地镜像

本文需要制作镜像，是因为需要安装新的 connections，如果不需要可以忽略本步。

Dockerfile

```
FROM apache/airflow:2.9.0
COPY requirements.txt /
RUN pip install --no-cache-dir "apache-airflow==${AIRFLOW_VERSION}" -r /requirements.txt
```

requirements.txt

```
apache-airflow-providers-apache-hive
apache-airflow-providers-apache-kafka
#apache-airflow-providers-apache-hdfs
apache-airflow-providers-elasticsearch
```

在该目录下执行`docker build -t trumandu/airflow:2.9.0 .`

### 启动服务

下载 docker-compose.yaml.

```
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.9.0/docker-compose.yaml'
```

修改基础镜像为以上重新制作的。

```
---
x-airflow-common: &airflow-common
  # In order to add custom dependencies or upgrade provider packages you can use your extended image.
  # Comment the image line, place your Dockerfile in the directory where you placed the docker-compose.yaml
  # and uncomment the "build" line below, Then run `docker-compose build` to build the images.
  image: ${AIRFLOW_IMAGE_NAME:-trumandu/airflow:2.9.0}
  # build: .
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ""
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true"
    AIRFLOW__CORE__LOAD_EXAMPLES: "true"
    AIRFLOW__API__AUTH_BACKENDS: "airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session"
    AIRFLOW__CORE__TEST_CONNECTION: "Enabled"
    # yamllint disable rule:line-length
    # Use simple http server on scheduler for health checks
    # See https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html#scheduler-health-check-server
    # yamllint enable rule:line-length
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: "true"
    # WARNING: Use _PIP_ADDITIONAL_REQUIREMENTS option ONLY for a quick checks
    # for other purpose (development, test and especially production usage) build/extend Airflow image.
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
    # The following line can be used to set a custom config file, stored in the local config folder
    # If you want to use it, outcomment it and replace airflow.cfg with the name of your config file
    # AIRFLOW_CONFIG: '/opt/airflow/config/airflow.cfg'
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on: &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always

  redis:
    image: redis:latest
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 50
      start_period: 30s
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-worker:
    <<: *airflow-common
    command: celery worker
    healthcheck:
      # yamllint disable rule:line-length
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.providers.celery.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}" || celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    environment:
      <<: *airflow-common-env
      # Required to handle warm shutdown of the celery workers properly
      # See https://airflow.apache.org/docs/docker-stack/entrypoint.html#signal-propagation
      DUMB_INIT_SETSID: "0"
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"',
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    # yamllint disable rule:line-length
    command:
      - -c
      - |
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg * 10 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    # yamllint enable rule:line-length
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: "true"
      _AIRFLOW_WWW_USER_CREATE: "true"
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
      _PIP_ADDITIONAL_REQUIREMENTS: ""
    user: "0:0"
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}:/sources

  airflow-cli:
    <<: *airflow-common
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
    command:
      - bash
      - -c
      - airflow

  # You can enable flower by adding "--profile flower" option e.g. docker-compose --profile flower up
  # or by explicitly targeted on the command line e.g. docker-compose up flower.
  # See: https://docs.docker.com/compose/profiles/
  flower:
    <<: *airflow-common
    command: celery flower
    profiles:
      - flower
    ports:
      - "5555:5555"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

volumes:
  postgres-db-volume:

```

初始化数据库

```
docker compose up airflow-init
```

运行 airflow

```
docker compose up
```

清理环境

```
docker compose down --volumes --remove-orphans
```

## airflow hive dag

### 创建 hive connection

![Img](http://image.trumandu.top/yank-note-picgo-img-20240413103845.png)
![Img](http://image.trumandu.top/yank-note-picgo-img-20240413104001.png)
如果 test 按钮无法点击可以通过如下两个办法：

1. 配置文件增加`test_connection = Enabled`

```
[core]
test_connection = Enabled
```

2. docker env 中增加`AIRFLOW__CORE__TEST_CONNECTION: "Enabled"`

### 创建 dag

在 dags 目录下创建 py 文件，airflow 会定时扫描文件，检测文件是否合法，如果合法，会在界面上看到响应的 dag,默认情况下，需要我们手动启用调度新增的 dag。

HiveServer2Hook,测试通过,更多 HiveServer2Hook 用法[详见地址](https://airflow.apache.org/docs/apache-airflow-providers-apache-hive/stable/_api/airflow/providers/apache/hive/hooks/hive/index.html#airflow.providers.apache.hive.hooks.hive.HiveServer2Hook)

```
from airflow import DAG
from airflow.utils.dates import days_ago
from airflow.decorators import task
from airflow.hooks import hive_hooks


with DAG(
    'hive',
    description='A simple hive DAG',
    schedule_interval='0/5 * * * *',
    start_date=days_ago(1),
) as dag:

    @task()
    def desc_hive():
        hh = hive_hooks.HiveServer2Hook(hiveserver2_conn_id='hive2',schema ='ecbd')
        sql='show databases'
        print(hh.get_records(sql))

    @task()
    def begain():
        print("hive begain job!")

    @task()
    def end():
        print("hive job finished success!")

    begain() >> desc_hive() >> end()
```

HiveOperator 测试未通过，错误信息：`PermissionError: [Errno 13] Permission denied: 'hive'` 测试需要在镜像中增加 hive client 脚本。未继续尝试

```
from airflow import DAG
from airflow.utils.dates import days_ago
from airflow.decorators import task
from airflow.providers.apache.hive.operators.hive import HiveOperator


with DAG(
    'show_databases',
    description='A simple hive DAG',
    schedule_interval='0/5 * * * *',
    start_date=days_ago(1),
) as dag:

    load_to_hive = HiveOperator(
        task_id=f"load_to_hive",
        hive_cli_conn_id="hive-cli",
        hql=(
            f"show databases"
        ),
    )

    @task()
    def begain():
        print("hive begain job!")

    @task()
    def end():
        print("hive job finished success!")

    begain() >> load_to_hive >> end()
```

HiveCliHook 测试未通过，错误信息：`PermissionError: [Errno 13] Permission denied: 'hive'` 测试需要在镜像中增加 hive client 脚本。未继续尝试

```
from airflow import DAG
from airflow.utils.dates import days_ago
from airflow.decorators import task
from airflow.hooks import hive_hooks


with DAG(
    'hive_cli_hooks',
    description='A simple hive DAG',
    schedule_interval='0/5 * * * *',
    start_date=days_ago(0),
) as dag:

    @task()
    def desc_hive():
        hh = hive_hooks.HiveCliHook(hive_cli_conn_id='hive-cli')
        sql='show databases'
        print(hh.run_cli(sql))

    @task()
    def begain():
        print("hive begain job!")

    @task()
    def end():
        print("hive job finished success!")

    begain() >> desc_hive() >> end()
```

### 手动执行 job

![Img](http://image.trumandu.top/yank-note-picgo-img-20240413103739.png)
![Img](http://image.trumandu.top/yank-note-picgo-img-20240413103819.png)

## 参考

1. [Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)
