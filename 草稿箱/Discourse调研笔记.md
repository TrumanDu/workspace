# Discourse 调研笔记

最近一个月的工作都集中在公司要搭建一个 Gamer Community 上，经过多个角度，多种开源，商业方案对比，最终选择了 discourse。

多说一句对于一个论坛技术选型，以前没有这方面的储备，最终的选择也是 AI 推荐的。领导下发任务用 AI 拆分，作为员工的我们，利用 AI 完成我们的方案调研和工作汇报，完美闭环！这么想，工资就白嫖来的！

## 部署

由于官方社区没有 docker 镜像，快速部署会有一点点麻烦，目前我推荐的 Discourse 有两种方式：

1. [官方社区方案](https://github.com/discourse/discourse_docker)
2. VMware 旗下[bitnami/discourse](https://hub.docker.com/r/bitnami/discourse)

最快速的方案推荐 bitnami，一个 docker-compose.yml 一条命令就可以跑起来。

```
services:
  postgresql:
    image: bitnami/postgresql:latest
    volumes:
      - "postgresql_data:/bitnami/postgresql"
    environment:
      # ALLOW_EMPTY_PASSWORD is recommended only for development.
      - ALLOW_EMPTY_PASSWORD=yes
      - POSTGRESQL_USERNAME=bn_discourse
      - POSTGRESQL_DATABASE=bitnami_discourse
  redis:
    image: bitnami/redis:latest
    environment:
      # ALLOW_EMPTY_PASSWORD is recommended only for development.
      - ALLOW_EMPTY_PASSWORD=yes
    volumes:
      - "redis_data:/bitnami/redis"
  discourse:
    image: bitnami/discourse:3
    ports:
      - "80:3000"
    volumes:
      - "discourse_data:/bitnami/discourse"
    depends_on:
      - postgresql
      - redis
    environment:
      # ALLOW_EMPTY_PASSWORD is recommended only for development.
      - ALLOW_EMPTY_PASSWORD=yes
      - DISCOURSE_HOST=127.0.0.1
      - DISCOURSE_DATABASE_HOST=postgresql
      - DISCOURSE_DATABASE_PORT_NUMBER=5432
      - DISCOURSE_DATABASE_USER=bn_discourse
      - DISCOURSE_DATABASE_NAME=bitnami_discourse
      - DISCOURSE_REDIS_HOST=redis
      - DISCOURSE_REDIS_PORT_NUMBER=6379
  sidekiq:
    image: bitnami/discourse:3
    depends_on:
      - discourse
    volumes:
      - "sidekiq_data:/bitnami/discourse"
    command: /opt/bitnami/scripts/discourse-sidekiq/run.sh
    environment:
      # ALLOW_EMPTY_PASSWORD is recommended only for development.
      - ALLOW_EMPTY_PASSWORD=yes
      - DISCOURSE_HOST=127.0.0.1
      - DISCOURSE_DATABASE_HOST=postgresql
      - DISCOURSE_DATABASE_PORT_NUMBER=5432
      - DISCOURSE_DATABASE_USER=bn_discourse
      - DISCOURSE_DATABASE_NAME=bitnami_discourse
      - DISCOURSE_REDIS_HOST=redis
      - DISCOURSE_REDIS_PORT_NUMBER=6379
volumes:
  postgresql_data:
    driver: local
  redis_data:
    driver: local
  discourse_data:
    driver: local
  sidekiq_data:
    driver: local
```

启动命令：`docker compose up -d`
删除容器和数据：`docker compose down --volumes`

## bitnami/discourse 使用

文档可以查看[ bitnami/discourse](https://hub.docker.com/r/bitnami/discourse)

所有操作都需要在工作目录，开始前需要提前配置代理等信息

```
cd /opt/bitnami/discourse
git config --global http.sslVerify false
export http_proxy=http://xxxxxxxxx:1080
export https_proxy=http://xxxxxxxxx:1080
```

### 安装插件

```
RAILS_ENV=production bundle exec rake plugin:install repo=https://github.com/discourse/discourse-oauth2-basic
RAILS_ENV=production bundle exec rake assets:precompile
```

### 安装主题

```
echo "redditish:
  url: https://github.com/discourse/discourse-redditish-theme
  add_to_all_themes: true" | tee my_theme.yml > /dev/null
RAILS_ENV=production bundle exec rake themes:install > my_theme.yml
RAILS_ENV=production bundle exec rake themes:update assets:precompile
```

有可能安装失败，原因未知。

## 主题开发

可以通过本地开发实时渲染更新到网站上，这种方式可以大大提高主题开发效率。具体文档详见：https://meta.discourse.org/t/install-the-discourse-theme-cli-console-app-to-help-you-build-themes/82950/1

通过 discourse_theme 命令可以将本地开发好的主题直接上传到网站上，同时还可以实时渲染看到更改效果！

1. 首先安装 Ruby,通过[Ruby Installer](https://rubyinstaller.org/)很容易安装
2. 然后通过命令安装 discourse cli `gem install discourse_theme`
3. 使用 cli
   例如：创建`discourse_theme new YOUR_DIR_NAME`
   实时渲染 `discourse_theme watch YOUR_DIR_NAME`

.discourse_theme 配置信息可以参考我本地：

```
---
C:/Users/truman/Desktop/discourse-redditish-theme:
  url: https://gamertest.xxxx.org
  components: none
  theme_id: 16
api_keys:
  https://gamertest.xxxx.org:
  xxxxxxxxxxxxx
```

使用命令：`discourse_theme watch .\discourse-redditish-theme\`

![Img](/images/Discourse调研笔记.md/img-20250325144749.png)

## API

查询所有的 Category

```
https://{host}/categories.json
```

查询 subcatefory

```
https://{host}/c/{id}.json
https://{host}/c/10.json
```

查询网站信息

```
https://{host}/about.json
```

## 网站设置

1.删除顶部 getting started
![Img](/images/Discourse调研笔记.md/img-20250326100823.png)
