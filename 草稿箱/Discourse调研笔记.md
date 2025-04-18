# Discourse 调研笔记

最近一个月的工作都集中在公司要搭建一个 Gamer Community 上，经过多个角度，多种开源，商业方案对比，最终选择了 discourse。

多说一句对于一个论坛技术选型，以前没有这方面的储备，最终的选择也是 AI 推荐的。领导下发任务用 AI 拆分，作为员工的我们，利用 AI 完成我们的方案调研和工作汇报，完美闭环！这么想，工资就白嫖来的！

## 部署

由于官方社区没有 docker 镜像，快速部署会有一点点麻烦，目前我推荐的 Discourse 有两种方式：

1. [官方社区方案](https://github.com/discourse/discourse_docker)
2. VMware 旗下[bitnami/discourse](https://hub.docker.com/r/bitnami/discourse)

### bitnami/discourse 部署

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

### 官方部署方案

```
sudo -s
git clone https://github.com/discourse/discourse_docker.git /var/discourse
cd /var/discourse
chmod 700 containers
```

安装向导，初始化配置文件

```
./discourse-setup
```

基于模板为配置启动一个容器

```
./launcher bootstrap app
```

## 官方 discourse 使用

```
Usage: launcher COMMAND CONFIG [--skip-prereqs] [--docker-args STRING]
Commands:
    start:       Start/initialize a container
    stop:        Stop a running container
    restart:     Restart a container
    destroy:     Stop and remove a container
    enter:       Open a shell to run commands inside the container
    logs:        View the Docker logs for a container
    bootstrap:   Bootstrap a container for the config based on a template
    run:         Run the given command with the config in the context of the last bootstrapped image
    rebuild:     Rebuild a container (destroy old, bootstrap, start new)
    cleanup:     Remove all containers that have stopped for > 24 hours
    start-cmd:   Generate docker command used to start container

Options:
    --skip-prereqs             Don't check launcher prerequisites
    --docker-args              Extra arguments to pass when running docker
    --skip-mac-address         Don't assign a mac address
    --run-image                Override the image used for running the container

```

重新构建一个容器

```
./launcher rebuild app

```

启动/停止应用

```
./launcher start app
./launcher stop app
```

进入容器

```
./launcher enter app
```

创建管理员账户

```
rake admin:create
```

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

## API

API 文档 https://docs.discourse.org/

Ruby API https://github.com/discourse/discourse_api

逆向 API https://meta.discourse.org/t/reverse-engineer-the-discourse-api/20576?silent=true

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

2.Summarize this topic button

这个功能默认是存在的，只是需要达到一个计算分数，具体详见：https://meta.discourse.org/t/summarize-this-topic-button/132790/17
![Img](/images/Discourse调研笔记.md/img-20250418164101.png)

## 预安装插件列表

| 名称                             | 文档                                                                 | 仓库                                                          | 是否安装 |
| :------------------------------- | :------------------------------------------------------------------- | :------------------------------------------------------------ | -------- |
| Discourse OAuth2 Basic           | https://meta.discourse.org/t/discourse-oauth2-basic/33879            | https://github.com/discourse/discourse-oauth2-basic           | yes      |
| Discourse Restricted Replies     | https://meta.discourse.org/t/discourse-restricted-replies/131343/1   | https://github.com/discourse/discourse-restricted-replies     | yes      |
| Discourse Category Experts       | https://meta.discourse.org/t/discourse-category-experts/190814       | https://github.com/discourse/discourse-category-experts       | yes      |
| Discourse AI                     | https://meta.discourse.org/t/discourse-ai/259214                     | https://github.com/discourse/discourse-ai                     | no       |
| Discourse Reactions              | https://meta.discourse.org/t/discourse-reactions/183261              | https://github.com/discourse/discourse-reactions              | yes      |
| Discourse Surveys                | https://meta.discourse.org/t/discourse-surveys/308679                | https://github.com/discourse/discourse-surveys                | yes      |
| Discourse Saved Searches         | https://meta.discourse.org/t/discourse-saved-searches/67901          | https://github.com/discourse/discourse-saved-searches         | yes      |
| Discourse Follow                 | https://meta.discourse.org/t/discourse-follow/110579                 | https://github.com/discourse/discourse-follow                 | yes      |
| Discourse Checklist              | https://meta.discourse.org/t/discourse-checklist/36362               | 默认已支持                                                    | --       |
| Discourse Subscriptions          | https://meta.discourse.org/t/discourse-subscriptions/140818          | https://github.com/discourse/discourse-subscriptions          | yes      |
| Discourse Signatures             | https://meta.discourse.org/t/discourse-signatures/42263              | https://github.com/discourse/discourse-signatures             | yes      |
| Discourse Cakeday                | https://meta.discourse.org/t/discourse-cakeday/38447                 | https://github.com/discourse/discourse-cakeday                | yes      |
| Discourse Tooltips               | https://meta.discourse.org/t/discourse-tooltips/69304                | https://github.com/discourse/discourse-tooltips               | yes      |
| Landing Pages Blog               | https://meta.discourse.org/t/landing-pages-blog/190390               | https://github.com/paviliondev/discourse-landing-pages        | --       |
| Discourse Newsletter Integration | https://meta.discourse.org/t/discourse-newsletter-integration/275509 | https://github.com/discourse/discourse-newsletter-integration | yes      |
| Discourse Yearly Review          | https://meta.discourse.org/t/discourse-yearly-review/105713          | https://github.com/discourse/discourse-yearly-review          | yes      |
| Discourse Translator             | https://meta.discourse.org/t/discourse-translator/32630              | https://github.com/discourse/discourse-translator             | yes      |
| discourse-user-notes             |                                                                      | https://github.com/discourse/discourse-user-notes             | yes      |
| discourse-data-explorer          | https://meta.discourse.org/t/discourse-data-explorer/32566           | https://github.com/discourse/discourse-data-explorer          | -------- |

## 数据库操作

变更管理员邮箱

```
SELECT * FROM users WHERE admin = true;


SELECT id, email FROM user_emails WHERE user_id = 1;--user@example.com


UPDATE user_emails SET email = 'user@example.com', updated_at = NOW() WHERE user_id = 1;

```

## 备份

1.修改备份包内容

```
gzip gamer-community-2025-04-09-052308-v20250321143553-1.tar.gz
tar -zczvf gamer-community-2025-04-09-052308-v20250321143553-1.tar.gz ./*
```

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

### components 开发

可以参考：https://github.com/discourse/discourse-profile-custom-link

`settings.yml` 用来设置配置信息

`common/common.scss` 设置公共样式

`javascripts/discourse/connectors` 设置插槽入口

`locales/en.yml` 设置国际化

`javascripts/discourse/components` 用来添加前端组件

举个例子，这里开发一个调用后端 api 获取相应数据。

```
export default class ProfileCustomLink extends Component {
  @service site;
  @service api;

  @tracked customLinkUrl;
  @tracked customLinkFieldId;
  @tracked showCustomLink = false;
  @tracked user = this.args.model.username;
  @tracked userId = this.args.model.id;
  @tracked userFields = this.args.model.user_fields;

  constructor() {
    super(...arguments);
    console.info("[Custom Profile Link] model", this.args.model);
    this.fetchExternalIds().then((oauth2BasicId) => {
      const url = settings.profile_custom_link_prefix + (oauth2BasicId == null ? '' : oauth2BasicId);
      this.customLinkUrl = url;
      this.showCustomLink = true;
    });
  }

  async fetchExternalIds() {
    try {
      const response = await ajax(`/admin/users/${this.userId}.json`, {
        headers: {
          "Api-Key": settings.profile_custom_link_api_key,
          "Api-Username": settings.profile_custom_link_api_username
        }
      });
      const oauth2BasicId = response.external_ids?.oauth2_basic;
      return oauth2BasicId
    } catch (error) {
      console.error("[Custom Profile Link] Error fetching external IDs:", error);
    }
  }
}

```

## 插件开发

插件和主题的 components 有点类似，但部分使用又有点区别，可以首先参考[discourse-plugin-skeleton](https://github.com/discourse/discourse-plugin-skeleton) 骨架项目创建一个插件项目。

开发文档：[Developing Discourse Plugins - Part 1 - Create a basic plugin](https://meta.discourse.org/t/developing-discourse-plugins-part-1-create-a-basic-plugin/30515?silent=true)

以下来解释一下项目目录：
```
.                                                                                                                                                                                                               
├── Gemfile
├── Gemfile.lock
├── LICENSE
├── README.md
├── app
│   └── controllers # 增加后端API 入口
│       └── discourse_profile
│           └── profile_controller.rb
├── assets
│   ├── javascripts
│   │   └── discourse
│   │       ├── components # 前端组件
│   │       │   └── profile-link.js
│   │       ├── connectors # 页面注入槽点
│   │       │   ├── user-card-post-names # 槽点
│   │       │   │   └── profile-link.hbs
│   │       │   └── user-location-and-website # 槽点
│   │       │       └── profile-link.hbs
│   │       └── templates
│   │           └── components
│   │               └── profile-link.hbs # 页面模板文件
│   └── stylesheets
│       └── common
│           └── profile.scss
├── config
│   ├── locales
│   │   └── client.en.yml # 国际化
│   ├── routes.rb # 路由配置
│   └── settings.yml # 配置参数
├── db
├── eslint.config.mjs
├── lib
│   └── profile_link
│       └── engine.rb
├── package.json
├── plugin.rb    # 插件注册和配置的入口
├── pnpm-lock.yaml
├── spec
│   └── system
│       └── core_features_spec.rb
├── stylelint.config.mjs
└── test
    └── javascripts
```



