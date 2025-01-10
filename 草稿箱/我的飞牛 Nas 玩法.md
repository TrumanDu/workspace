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

### UptimeKuma

### 浏览器

### tao-sync

https://github.com/dr34m-cn/taosync

### alist

### 夸克网盘自动转存

https://github.com/Cp0204/quark-auto-save

### IPTV

doube-ipv
