# 原力计划-从零实现最简 http 服务器

> 周末了，你带你的破电脑回到家并不能给你带来任何实质性作用，朋友们兜里掏出一大把钱吃喝玩乐，你默默的在家里摆弄你的破烂 java，spring 全家桶，srpingcloud 看了多少遍算法。亲戚朋友吃饭问你收获了什么，你说我装了个虚拟机，把各个工具都玩了一遍，亲戚们懵逼了，你还在心里默默嘲笑他们，笑他们不懂你的自动注入，不懂你的 10 层代理、不懂你的流量混淆，也笑他们连个复杂点的密码都记不住。你父母的同事都在说自己的子女一年的收获，儿子买了个房，女儿买了个车，姑娘升职加薪了，你的父母默默无言，说我的儿子搞了个破电脑，开起来嗡嗡响、家里电表走得越来越快了。

我就是那个段子里的<u>大冤种</u>，正文开始前自我调侃一下。

随着工作的年限的增长，越来越难写一些基础代码，总感觉自己的底子打的比较差，工作中写不了硬核代码，那我就用业余时间完成吧！我也知道网上代码一大堆，及时自己实现了也不见得有什么收获，就当学习，磨一磨生锈的刀。

若干年前在一个 java 项目中看到用纯 java 手撕前端代码，当时觉的好神奇，也大概只有功力深厚，大牛才能写出这样的代码，我等菜鸡只能写写 CURD,也许是那个时候埋下的种子吧，tomcat 没有精力写，但写一个简单的 http 服务器，让自己的网站跑在这个服务器上，想想也是美滋滋，什么 tomcat,什么 nginx,自己的服务器自己来写。

**HTTP 服务器目标：让自己的网站跑在拥有自主知识产权的静态服务器上。**

开始前要搞懂 http 协议。

## HTTP 协议

HTTP 协议使用客户端-服务器模型，其中客户端发送一个 HTTP 请求到服务器，服务器则返回一个 HTTP 响应。HTTP 请求由请求方法（GET、POST、PUT、DELETE 等）和请求头（包含请求信息的元数据）组成，而 HTTP 响应由状态码（描述服务器对请求的响应）、响应头和响应正文（包含实际数据）组成。

以下是一个使用 HTTP 协议进行请求和响应的简单 demo：

1.  客户端向服务器发送 GET 请求：

    ```
    GET /hello.txt HTTP/1.1
    Host: example.com
    User-Agent: Mozilla/5.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    ```

2.  服务器向客户端发送响应：
    ```
    HTTP/1.1 200 OK
    Content-Type: text/plain
    Content-Length: 12

    Hello World!
    ```

在这个例子中，服务器返回一个 HTTP/1.1 200 OK 响应码，表示请求成功。响应头中包含了一些附加的信息，例如返回的数据类型和数据长度。响应正文中包含了实际的数据，即 Hello World!。

java 代码实现如下：

```
public class SimpleHttp {
    private static int port;
    public static void main(String[] args) throws IOException {
        int port = 8080;
        ServerSocket ss = new ServerSocket(port);
        System.out.println("Simple http server started on port " + port);

        while (true) {
            Socket socket = ss.accept();
            System.out.println("New client connected");

            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String line;
            while ((line = in.readLine()) != null) {
                if (line.isEmpty()) {
                    break;
                }
                System.out.println(line);
            }
            String response = """
                    HTTP/1.1 200 OK

                    Hello, World!
                    """;
            OutputStream outputStream = socket.getOutputStream();
            outputStream.write(response.getBytes());
            outputStream.flush();
            outputStream.close();
        }
    }
}
```

以上代码只是实现很简单 demo,还远远不足跑起来我的个人网站。

## 实现静态服务器

基于以上的代码骨架，我们来丰富一下细节：

```
public void start(int port) throws IOException {
        ServerSocket ss = new ServerSocket(port);
        System.out.println("Simple http server started on port " + port);
        HttpResponse response;
        while (true) {
            Socket socket = ss.accept();
            try {
                HttpRequest request = handleRequest(socket);
                response = handleResponse(request, socket);
            } catch (Exception e) {
                response = handle5xx();
            }
            response(response, socket);
        }
    }
```

对于 request 处理，其实是为了获得请求的文件路径及 header 等其他信息,

```
    private HttpRequest handleRequest(Socket socket) throws IOException {
        BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        String line;
        List<String> rawHeaders = new ArrayList<>(64);
        String firstLine = null;
        while ((line = in.readLine()) != null) {
            if (line.isEmpty()) {
                break;
            }
            if (firstLine == null) {
                firstLine = line;
            } else {
                rawHeaders.add(line);
            }
        }
        if (firstLine == null) {
            return null;
        }
        String[] array = firstLine.split(" ", 3);
        HttpRequest httpRequest = new HttpRequest(array[0], array[1], array[2]);
        httpRequest.setHeaders(rawHeaders);
        return httpRequest;
    }
```

接下来我们实现一下 response

```
    private HttpResponse handleResponse(HttpRequest request, Socket socket) throws IOException {
        String path = request.getPath();
        HttpResponse response;
        try {
            if ("/".equalsIgnoreCase(path) || path.length() == 0) {
                path = "/index.html";
            } else if (path.indexOf(".") < 0) {
                path = path + ".html";
            }
            boolean flag = ResourcesFileUtil.isExistFile(path);
            if (!flag) {
                path = request.getPath() + "/index.html";
                flag = ResourcesFileUtil.isExistFile(path);
            }
            if (!flag) {
                response = handle404();
            } else {
                response = handleOk(path);
            }
        } catch (Exception e) {
            response = handle5xx();
        }
        return response;
    }
```

这里有一些特殊逻辑处理，主要是处理没有文件后缀名`.html`，或者找该路径下的`index.html`

然后根据文件是否存在进行相应的处理，如果找不到文件，使用`handle404`方法来处理，该文件存在的话，使用`handleOk`来处理。

```
    private HttpResponse handle404() throws IOException {
        String body = "Page not fond!";
        HttpResponse response = new HttpResponse(404, body.getBytes());
        Headers headers = new Headers();
        headers.addHeader("Content-Type", ContextType.getContextType("html"));
        headers.addHeader("Content-Length", body.getBytes().length + "");
        headers.addHeader("Connection", "Close");
        response.setHeaders(headers);
        return response;
    }
```

```
    private HttpResponse handleOk(String path) throws IOException {
        String suffix = "html";
        if (path.lastIndexOf(".") > 0) {
            suffix = path.substring(path.lastIndexOf(".") + 1, path.length());
        }
        byte[] body = ResourcesFileUtil.getResource(path);
        HttpResponse response = new HttpResponse(200, body);
        Headers headers = new Headers();
        headers.addHeader("Content-Type", ContextType.getContextType(suffix));
        headers.addHeader("Content-Length", "" + body.length);
        response.setHeaders(headers);
        return response;
    }
```

本质上就是封装生成 http 协议需要的文本内容，对于不同文件类型，要返回不同的`Content-Type`。

```
public static String getContextType(String suffix) {
        assert suffix != null;
        switch (suffix.toLowerCase()) {
            case "html":
                return "text/html; charset=utf-8";
            case "css":
                return "text/css; charset=utf-8";
            case "js":
                return "text/javascript; charset=utf-8";
            case "jpeg":
            case "jpg":
                return "image/jpeg";
            case "png":
                return "image/png";
            case "gif":
                return "image/gif";
            case "ico":
                return "image/x-icon";
            case "json":
                return "application/json; charset=utf-8";
            default:
                return "text/plain; charset=utf-8";
        }
    }
```

生成好内容以后，将内容返回到客户端：

```
 private void response(HttpResponse response, Socket socket) throws IOException {
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(response.getFirstLine());
        outputStream.write(response.getHeaders().toString().getBytes());
        outputStream.write("\r\n".getBytes());
        outputStream.write(response.getBody());
        outputStream.flush();
        outputStream.close();
    }
```

关于其他代码就省略了，主要是封装和复用吧，具体代码看源码吧。

访问 http://localhost:8080/ 这下就能看到我的网站啦！纯纯的高科技，自主可控！

```
Simple http server started on port 8080
2023-03-26 22:42:32 INFO GET /  Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36
```

![Img](http://static.trumandu.top/yank-note-picgo-img-20230326224311.png)

## NIO 实现静态服务器

以上实现已经可以完美的运行我的网站了，美中不足的是服务器端是阻塞服务，性能较差，我们再基于 Java NIO 来个性能版，使用 Selector 和 ServerSocketChannel 实现多路复用静态服务器。

修改事件处理逻辑：

```
    public void start(int port) throws IOException {
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.bind(new InetSocketAddress(port));
        ssc.configureBlocking(false);
        System.out.println("Simple http server started on port " + port);

        Selector selector = Selector.open();
        ssc.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            int readNum = selector.select();
            if (readNum == 0) {
                continue;
            }
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey selectionKey = iterator.next();
                iterator.remove();
                //处理连接就绪事件
                if (selectionKey.isAcceptable()) {
                    ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();
                    SocketChannel socketChannel = channel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector, SelectionKey.OP_READ);
                } else if (selectionKey.isReadable()) {
                    handleRequest(selectionKey);
                    // 注册一个写事件，用来给客户端返回信息
                    selectionKey.interestOps(SelectionKey.OP_WRITE);
                } else if (selectionKey.isWritable()) {
                    handleResponse(selectionKey);
                }
            }
        }
    }
```

处理读请求：

```
    private void handleRequest(SelectionKey selectionKey) throws IOException {
        SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        socketChannel.read(buffer);
        byte[] requestBytes = buffer.array();
        int readBytes;
        while ((readBytes = socketChannel.read(buffer)) != 0) {
            if (readBytes == -1) {
                // 如果读到了流的末尾，则表示连接已经断开
                break;
            }
            buffer.flip();
            int size = buffer.array().length;
            byte[] temp = new byte[requestBytes.length + size];
            System.arraycopy(requestBytes, 0, temp, 0, requestBytes.length);
            System.arraycopy(buffer.array(), 0, temp, requestBytes.length, size);
            requestBytes = temp;
            buffer.clear();
        }
        String context = new String(requestBytes);

        String[] lines = context.split("\r\n");
        String firstLine = null;
        List<String> rawHeaders = new ArrayList<>(64);
        for (int i = 0; i < lines.length; i++) {
            String line = lines[i];
            if (i == 0) {
                firstLine = line;
            } else {
                rawHeaders.add(line);
            }
        }
        if (firstLine == null) {
            return;
        }

        String[] array = firstLine.split(" ", 3);
        HttpRequest httpRequest = new HttpRequest(array[0], array[1], array[2]);
        httpRequest.setHeaders(rawHeaders);
        selectionKey.attach(httpRequest);
    }
```

处理写请求：

```
    private void handleResponse(SelectionKey selectionKey) throws IOException {
        SocketChannel channel = (SocketChannel) selectionKey.channel();
        HttpRequest request = (HttpRequest) selectionKey.attachment();
        String path = request.getPath();
        HttpResponse response;
        try {
            if ("/".equalsIgnoreCase(path) || path.length() == 0) {
                path = "/index.html";
            } else if (path.indexOf(".") < 0) {
                path = path + ".html";
            }
            boolean flag = ResourcesFileUtil.isExistFile(path);
            if (!flag) {
                path = request.getPath() + "/index.html";
                flag = ResourcesFileUtil.isExistFile(path);
            }
            if (!flag) {
                response = handle404();
            } else {
                response = handleOk(path);
            }
        } catch (Exception e) {
            response = handle5xx();
        }
        channel.write(ByteBuffer.wrap(response.getFirstLine()));
        channel.write(ByteBuffer.wrap(response.getHeaders().toString().getBytes()));
        channel.write(ByteBuffer.wrap("\r\n".getBytes()));
        channel.write(ByteBuffer.wrap(response.getBody()));
        channel.close();
    }
```

本文所有代码：[https://github.com/TrumanDu/the-force](https://github.com/TrumanDu/the-force/tree/main/simple-httpd)

## 说明

🙌 如果你阅读到这里，相信我们一定是**同道中人**，有任何想法，欢迎**私聊**我，微信号：**trumandu007**。
💡 如果你也是在西安地区从事 IT 相关工作，欢迎私信加入我建的 **『西安 IT 技术圈』**微信群,我们是一个什么样的群体？ [为什么要做『西安 IT 技术圈』](https://mp.weixin.qq.com/s?__biz=MzI4NTMwNTQ5Mg==&mid=2247483684&idx=1&sn=4c1f96c16463601a7e220a06649f4cd3)。
👬🏻 朋友，都看到这了，确定不关注一下么 👇

![](http://static.trumandu.top/trumandu_wechat_2022.png)
