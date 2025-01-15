# 我的飞牛 Nas 玩法

中年男性三大爱好，钓鱼，摄影，**NAS**。 因为穷，只玩得起 NAS,还是垃圾佬玩法。

折腾的路上是真的其乐无穷，情绪价值也是价值，几百块钱带来的快乐是无法比拟的。之前在闲鱼上淘了蜗牛星际，装了黑裙，基本满足了自己文件备份的需求，很少用来折腾，总觉得费电，看电影我有其他方案，好像除了冷备文件外也没有更进一步折腾的需求。

最近一直被同事安利飞牛和 N100，在闲鱼摸索了几个月终于还是入手了，接下来说说我的折腾之路。

## 设备

最近 💗 入手一个 N100 小主机，火影 minit8Plus 迷你主机 n100 16+512。

![Img](https://static.trumandu.top/yank-note-picgo-img-20250109221733.png)

## 软件

### Bark 通知

先来简单介绍一下 Bark,**免费、轻量**！简单调用接口即可给自己的 iPhone 发送推送。
依赖苹果 APNs，及时、稳定、可靠,不会消耗设备的电量， 基于系统推送服务与推送扩展，APP 本体并不需要运行。隐私安全，可以通过一些方式确保包含作者本人在内的所有人都无法窃取你的隐私。

原理是作者自己开发了一个 APP,并且公布了自己的证书，借助苹果的 APNs, 通过 Rest 就可以发送通知了。

其实只需要下载 APP,就可以发送通知了，但为了更方便，安全就自己部署`bark-server`。bark-server 只是对 APNs 服务调用的封装，利用作者发布的证书，也可以自己写代码。为了直接使用，我选择部署 bark-server。

第一步：

```
docker run -dt --name bark -p 8080:8080 -v `pwd`/bark-data:/data finab/bark-server
```

第二步：注册设备（这一步也可以在 APP 中添加来完成）

```
GET http://192.168.0.110:8080/register?devicetoken={替换为自己的}

{
    "device_key": "xxxx",
    "device_token": "zzzzzzzzzz",
    "key": "xxxx"
}

```

第三步：发送测试

```
GET http://192.168.0.110:8080/xxxxx/推送内容?group=分组
```

如下为我给 UptimeKuma 配置的通知
![Img](https://static.trumandu.top/yank-note-picgo-img-20250109221124.png)

更多用法与文档详见：[https://bark.day.app/#/?id=bark](https://bark.day.app/#/?id=bark)

不用配置 bark-server，可以使用 python 代码直接调用 APNs，demo 如下：

```
import jwt
import time
import httpx
from pathlib import Path

# 配置环境变量
TOKEN_KEY_FILE_NAME = "./AuthKey_LH4T9V5U4R_5U8LBRXG3A.p8"  # 替换为你的 key 文件路径
DEVICE_TOKEN = "xxxxxx"  # 从 app 设置中复制的 DeviceToken

# 固定参数
TEAM_ID = "5U8LBRXG3A"
AUTH_KEY_ID = "LH4T9V5U4R"
TOPIC = "me.fin.bark"
APNS_HOST_NAME = "api.push.apple.com"


# 生成 JWT Token
def generate_auth_token():
    # 读取密钥文件
    private_key = Path(TOKEN_KEY_FILE_NAME).read_text()

    # 生成 JWT Header 和 Claims
    headers = {"alg": "ES256", "kid": AUTH_KEY_ID}
    payload = {"iss": TEAM_ID, "iat": int(time.time())}

    # 签名生成 Token
    token = jwt.encode(payload, private_key, algorithm="ES256", headers=headers)
    return token


# 发送推送
def send_push_notification(token):
    headers = {
        "apns-topic": TOPIC,
        "apns-push-type": "alert",
        "authorization": f"bearer {token}"
    }
    payload = {
        "aps": {
            "mutable-content": 1,
            "alert": {
                "title": "title",
                "body": "body"
            },
            "category": "myNotificationCategory",
            "sound": "minuet.caf"
        },
        "icon": "https://day.app/assets/images/avatar.jpg"
    }

    # 使用 httpx 发送请求
    url = f"https://{APNS_HOST_NAME}/3/device/{DEVICE_TOKEN}"
    with httpx.Client(http2=True) as client:
        response = client.post(url, json=payload, headers=headers)  # 示例 GET 请求
        if response.status_code == 200:
            print("推送成功！")
        else:
            print(f"推送失败，状态码: {response.status_code}")
            print(f"响应内容: {response.text}")


if __name__ == "__main__":
    # 生成 Token
    auth_token = generate_auth_token()

    # 发送推送通知
    send_push_notification(auth_token)
```

### UptimeKuma

https://github.com/louislam/uptime-kuma

一个易于使用的自托管网站健康状态监控开源项目

![Img](https://static.trumandu.top/yank-note-picgo-img-20250114161843.png)

### 浏览器

![浏览器](https://static.trumandu.top/yank-note-picgo-img-20250115224613.png)

在 nas 中跑浏览器的最大好处就是可以不用担心信息安全，不用担心公司记录你的浏览记录，安全摸鱼，可信摸鱼。我这里是在飞牛应用中心安装的。

### tao-sync

https://github.com/dr34m-cn/taosync

因为 Alist 中不支持定时同步功能，通过 taosync 可以实现定时同步功能。它的原理是调用 alist 的复制接口实现的。

![Img](https://static.trumandu.top/yank-note-picgo-img-20250115211240.png)

### alist

https://github.com/AlistGo/alist

一个支持多存储的文件列表/WebDAV 程序，支持百度，阿里云，夸克云，S3 等多种存储，基本弥补 Nas 对网盘支持不足的问题。

![Img](https://static.trumandu.top/yank-note-picgo-img-20250114162431.png)

### 夸克网盘自动转存

https://github.com/Cp0204/quark-auto-save

夸克网盘签到、自动转存、命名整理、发推送提醒和刷新媒体库一条龙

![Img](https://static.trumandu.top/yank-note-picgo-img-20250114160218.png)

### IPTV

https://hub.docker.com/r/doubebly/doube-itv

doube-ipv

### Sun-Panel

https://doc.sun-panel.top/zh_cn/

一个 NAS、服务器导航面板、简易 docker 管理器、Homepage、浏览器首页

飞牛部署的 docker 软件越来越多，标签收藏也越来越多，为了统一记录各个服务的地址，我就部署了 Sun-Panel，曾经以为我不需要导航面板，现在回想一下，也是不错的。

![Sun-Panel](https://static.trumandu.top/yank-note-picgo-img-20250115223415.png)
