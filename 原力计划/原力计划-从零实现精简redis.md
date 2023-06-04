# 原力计划-从零实现精简 redis

如何学习编程？

受[George Hotz](https://www.youtube.com/@geohotarchive/videos)的影响，比较赞同：Learn by doing，找个自己感兴趣的项目，直接开干，在过程中学习。

很早之前就接触 redis,惊叹作者的代码和设计，如果你想学习数据库或者 cache 系统，推荐你看一下 redis 源码，短小精悍，完美融合了各种数据结构，协议的设计也完美的符合简单哲学。

我想学一下 go 语言，同时还能考虑一下 redis 的设计，这就是这个项目的最初动力。

go cache just for learn redis design and golang

本文所有代码：[https://github.com/TrumanDu/the-force](https://github.com/TrumanDu/the-force/tree/main/gocache)

## 项目初始化

在项目构建期纠结了很久，不知道如何组织 go 项目目录，因为自己的做 java 开发的，自己只能借鉴开源的经验

以下是我开始这个项目前参考的链接：

1. [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
2. [How to Write Go Code](https://golang.org/doc/code.html)

### 构建自己的项目目录结构

project-layout 能告诉我目前社区流行的 go 项目都采用什么目录结构。
根据自己的想法，目前构建如下：

```
├─api //提供项目的api
├─build // 编译目录
│  ├─ci
│  └─package
├─cmd
│  └─gocache // 应用启动入口
├─configs // 应用配置
├─docs // 存放文档
├─init // 初始化
├─server
├─store
│  └─cache
└─tools // 工具类

```

### 开发

How to Write Go Code 能告诉我们如何使用 go mod,安装应用，导入包，测试

#### go mod

```
go mod init github.com/trumandu/gocache
```

### 更多学习 go 资源

1. [Effective Go](https://golang.org/doc/effective_go.html#introduction)
2. [高效的 Go 编程 Effective Go](https://learnku.com/docs/effective-go/2020)
3. [Ultimate Go study guide](https://github.com/hoanhan101/ultimate-go)

## 网路 IO 模型开发

redis 选用的单 Reactor 模型，虽然 go 编程模型对于 goroutine 创建属于轻量级的，比
线程耗的资源更低，但是一个 goroutine stack 也会占用 2k-8k。对于百万级连接，内存
占用也会很高，达到几十 G,为了更好的性能，和更贴近 redis 设计。我这边也采用 Reactor。

### Reactor 模型

![](http://static.trumandu.top/20230604184759.png)

Reactor 模型其实就是 IO 多路复用+池化技术。

多说一句：复用指的是复用了 1 个线程，一个线程可以同时处理多个 fd(文件描述符)

Reactor 架构模式允许事件驱动的应用通过多路分发的机制去处理来自不同客户端的多个请求。

### 实践

```
//Op op
type Op uint32

//Epoller Epoller
type Epoller struct {
	epfd        int
	connections map[int]net.Conn
}

//NewEpoller new epoller
func NewEpoller() (*Epoller, error) {

	epfd, err := unix.EpollCreate(1)
	if err != nil {
		return nil, err
	}
	epoller := &Epoller{epfd: epfd, connections: make(map[int]net.Conn)}
	return epoller, nil
}

func (epoller *Epoller) Close() error {
	return unix.Close(epoller.epfd)
}

func (epoller *Epoller) Add(conn net.Conn) error {
	fd := socketFD(conn)
	event := unix.EpollEvent{
		Events: unix.EPOLLIN | unix.EPOLLHUP,
		Fd:     int32(fd),
	}
	err := unix.EpollCtl(epoller.epfd, unix.EPOLL_CTL_ADD, fd, &event)

	if err != nil {
		return err
	}
	epoller.connections[fd] = conn
	return nil
}

func (epoller *Epoller) Remove(conn net.Conn) error {
	fd := socketFD(conn)
	err := unix.EpollCtl(epoller.epfd, unix.EPOLL_CTL_DEL, fd, nil)

	if err != nil {
		return err
	}
	delete(epoller.connections, fd)
	return nil
}

func (epoller *Epoller) Wait() ([]net.Conn, error) {
	events := make([]unix.EpollEvent, 10)
	n, err := unix.EpollWait(epoller.epfd, events, 10)
	if err != nil && err != unix.EINTR {
		return nil, err
	}
	var connections []net.Conn
	for i := 0; i < n; i++ {
		conn := epoller.connections[int(events[i].Fd)]
		connections = append(connections, conn)
	}
	return connections, nil
}

func socketFD(conn net.Conn) int {

	tcpConn := reflect.Indirect(reflect.ValueOf(conn)).FieldByName("conn")
	fdVal := tcpConn.FieldByName("fd")
	pfdVal := reflect.Indirect(fdVal).FieldByName("pfd")
	return int(pfdVal.FieldByName("Sysfd").Int())
}
```

处理网络请求
```go
func Run() {
	listen, err := net.Listen("tcp", ":"+port)
	if err != nil {
		log.Error("listen fail:", err)
		return
	}

	defer listen.Close()

	epoller, err := NewEpoller()

	if err != nil {
		panic(err)
	}

	go listenClientConnect(epoller, listen)

	for {
		connections, err := epoller.Wait()
		if err != nil {
			log.Error("epoll wait error:", err)
			continue
		}

		if len(connections) > 0 {
			coreProcess(epoller, connections)
		}
	}

}

func listenClientConnect(epoller *Epoller, listen net.Listener) {
	for {
		//阻塞等待用户链接
		conn, err := listen.Accept()
		if err != nil {
			log.Error(err)
			return
		}
		key := conn.RemoteAddr().String()
		clientsMap[key] = &clientConn{conn, conn.RemoteAddr().String(), NewRedisReader(conn), NewRedisWriter(conn)}
		epoller.Add(conn)
	}
}
```


### 参考

1. [百万 Go TCP 连接的思考: epoll 方式减少资源占用](https://colobu.com/2019/02/23/1m-go-tcp-connection/)
2. [smallnest/1m-go-tcp-server](https://github.com/smallnest/1m-go-tcp-server)
3. [epoll 多路复用-----epoll_create1()、epoll_ctl()、epoll_wait()](https://blog.csdn.net/displayMessage/article/details/81151646)
4. [Minimal viable epoll package for go](https://gist.github.com/ast/a41816345e94e065890440e87e41a219)

### Redis 协议解析

最新版是 RESP3（Redis 序列化协议），人类可读，使用简单而紧凑的格式进行序列化和解析，通过定义不同的数据类型和指令来表示各种操作。RESP3 协议支持双向通信，客户端可以向服务器发送命令并接收响应。

先定义类型：

```go
const (
	// TypeBlobString simple types
	TypeBlobString     = '$' // $<length>\r\n<bytes>\r\n
	TypeSimpleString   = '+' // +<string>\r\n
	TypeSimpleError    = '-' // -<string>\r\n
	TypeNumber         = ':' // :<number>\r\n
	TypeNull           = '_' // _\r\n
	TypeDouble         = ',' // ,<floating-point-number>\r\n
	TypeBoolean        = '#' // #t\r\n or #f\r\n
	TypeBlobError      = '!' // !<length>\r\n<bytes>\r\n
	TypeVerbatimString = '=' // =<length>\r\n<format(3 bytes):><bytes>\r\n
	TypeBigNumber      = '(' // (<big number>\n
	// TypeArray Aggregate data types
	TypeArray     = '*' // *<elements number>\r\n... numelements other types ...
	TypeMap       = '%' // %<elements number>\r\n... numelements key/value pair of other types ...
	TypeSet       = '~' // ~<elements number>\r\n... numelements other types ...
	TypeAttribute = '|' // |~<elements number>\r\n... numelements map type ...
	TypePush      = '>' // ><elements number>\r\n<first item is String>\r\n... numelements-1 other types ...
	// TypeStream special type
	TypeStream = "$EOF:" //
)
```

然后再定义 Reader 和 Writer

```go
// Reader struct
type RedisReader struct {
	*bufio.Reader
}

// NewRedisReader method
func NewRedisReader(reader io.Reader) *RedisReader {

	return &RedisReader{Reader: bufio.NewReaderSize(reader, defaultSize)}
}
type RedisWriter struct {
	*bufio.Writer
}

func NewRedisWriter(writer io.Writer) *RedisWriter {
	return &RedisWriter{bufio.NewWriterSize(writer, defaultSize)}
}
```

读取网络流中的不同类型的数据

```go
func (r *RedisReader) ReadValue() (*Value, error) {
	line, err := r.readLine()
	if err != nil {
		return nil, err
	}
	if len(line) < 3 {
		return nil, ErrInvalidSyntax
	}
	v := &Value{
		Type: line[0],
	}
	switch v.Type {
	case TypeSimpleString, TypeSimpleError:
		v.Str, err = r.readSimpleString(line)
		v.Size = int64(3) + int64(len(v.Str))
	case TypeNumber, TypeBoolean, TypeDouble, TypeBigNumber:
		// TODO 待实现
		v.Str, err = r.readSimpleString(line)
	case TypeBlobString, TypeBlobError:
		v.Str, v.Size, err = r.readBlobString(line)
	case TypeArray:
		v.Elems, v.Size, err = r.readArray(line)
	}
	return v, err
}
func (r *RedisReader) readSimpleString(line []byte) (string, error) {
	return string(line[1 : len(line)-2]), nil
}
func (r *RedisReader) readBlobString(line []byte) (string, int64, error) {
	count, index, err := r.getCount(line)
	if err != nil {
		return "", 0, err
	}
	buf := make([]byte, count+2)
	_, err = io.ReadFull(r, buf)
	if err != nil {
		return "", 0, err
	}
	return string(buf[:count]), int64(count) + int64(index+2), nil
}

func (r *RedisReader) readArray(line []byte) ([]*Value, int64, error) {
	count, index, err := r.getCount(line)
	byteSize := int64(index)
	if err != nil {
		return nil, 0, err
	}
	var values []*Value
	for i := 0; i < count; i++ {
		v, err := r.ReadValue()
		if err != nil {
			return nil, 0, err
		}
		byteSize = byteSize + v.Size
		values = append(values, v)
	}

	return values, byteSize, nil
}
```

服务器处理请求后写数据操作：

```go
func (w *RedisWriter) replyNull() []byte {
	bs := []byte{TypeBlobString, '-', '1'}
	bs = append(bs, CRLF...)
	return bs
}
func (w *RedisWriter) replyNumber(num int) []byte {
	bs := []byte{TypeNumber}
	my := []byte(strconv.Itoa(num) + CRLF)
	bs = append(bs, my...)
	return bs
}

func (w *RedisWriter) replyString(message string) []byte {
	bs := []byte{TypeSimpleString}
	my := []byte(message + CRLF)
	bs = append(bs, my...)
	return bs
}

func (w *RedisWriter) replyArray(messages []string) []byte {
	bs := []byte{TypeArray}
	my := []byte(strconv.Itoa(len(messages)) + CRLF)
	bs = append(bs, my...)

	for _, arg := range messages {
		bs = append(bs, TypeBlobString)
		str := []byte(strconv.Itoa(len(arg)) + CRLF + arg + CRLF)
		bs = append(bs, str...)
	}
	return bs
}


```

项目代码目前支持如下类型：

| Type          | Comment                                                  |
| ------------- | -------------------------------------------------------- |
| Array         | an ordered collection of N other types                   |
| Blob string   | binary safe strings                                      |
| Simple string | a space efficient non binary safe string                 |
| Simple error  | a space efficient non binary safe error code and message |
| Number        | an integer in the signed 64 bit range                    |
| Null          | RESP2.0 null                                             |

### 参考

1. [使用 Go 语言读写 Redis 协议 ](https://colobu.com/2019/04/16/Reading-and-Writing-Redis-Protocol-in-Go/)
2. [写 Redis RESP3 协议以及 Redis 6.0 客户端缓存](https://colobu.com/2019/08/09/read-and-write-redis-RESP3-protocol/)
3. [RESP3 specification](https://github.com/antirez/RESP3/blob/master/spec.md)

## Cache 实现

这里简单使用 map 来实现

```go
var cache = make(map[string]string)

func Get(key string) string {
	return cache[key]
}

func Del(key string) int {
	if _, ok := cache[key]; ok {
		delete(cache, key)
		return 1
	} else {
		return 0
	}
}

func Set(key string, val string) bool {
	cache[key] = val
	return true
}

func Exists(key string) int {
	if _, ok := cache[key]; ok {
		return 1
	} else {
		return 0
	}
}
```

## 核心处理逻辑

```go
func coreProcess(epoller *Epoller, connections []net.Conn) {
	// 1.多线程读取解析client发来的数据（命令和数据类型等）
	clientsReadSyncList := handleClientsWithPendingReadsUsingThreads(epoller, connections)
	// 2.单线程执行command
	responses := handleCommand((*clientsReadSyncList).list)
	if appendonly {
		if len(aofBuf) > 0 {
			appendAOF(aofBuf)
		}
	}
	// 3.多线程向client发送响应数据
	handleClientsWithPendingWritesUsingThreads(responses)
}

func handleClientsWithPendingReadsUsingThreads(epoller *Epoller, connections []net.Conn) *SyncList {
	funcs := make([]func(), len(connections))
	clientsReadSyncList := NewSyncList()
	for i := 0; i < len(connections); i++ {
		conn := connections[i]
		id := conn.RemoteAddr().String()
		funcs[i] = func() {
			clientConn := clientsMap[id]
			value, err1 := clientConn.rd.ReadValue()
			if err1 != nil {
				if err := epoller.Remove(conn); err != nil {
					log.Error("failed to remove :", err)
				}
				conn.Close()
				return
			}

			data := &readData{id, value}
			clientsReadSyncList.Add(data)
		}
	}
	ioPool.SyncRun(funcs)
	return clientsReadSyncList
}

func handleCommand(clientsRead *list.List) (responseList *list.List) {
	responseList = list.New()
	aofBuf = make([]byte, 0)
	for e := clientsRead.Front(); e != nil; e = e.Next() {

		data := e.Value.(*readData)
		value := data.value
		id := data.id
		wt := clientsMap[id].wt
		var responseBytes []byte
		command := ""
		switch value.Type {

		case TypeSimpleError:
			log.Error(value.Err)
		case TypeSimpleString:
			log.Error("wait todo...")
		case TypeArray:
			array := value.Elems
			command = strings.ToLower(array[0].Str)
			switch command {
			case "ping":
				responseBytes = wt.replyString("PONG")
			case "quit":
				responseBytes = wt.replyString("OK")
			case "set":
				if len(array) < 3 {
					responseBytes = wt.replyInvalidSyntax()
				} else {
					cache.Set(array[1].Str, array[2].Str)
					responseBytes = wt.replyString("OK")
				}

			case "exists":
				if len(array) < 2 {
					responseBytes = wt.replyInvalidSyntax()
				} else {
					responseBytes = wt.replyNumber(cache.Exists(array[1].Str))
				}
			case "get":
				if len(array) < 2 {
					responseBytes = wt.replyInvalidSyntax()
				} else {
					data := cache.Get(array[1].Str)
					if data != "" {
						responseBytes = wt.replyString(data)
					} else {
						responseBytes = wt.replyNull()
					}

				}
			case "del":
				if len(array) < 2 {
					responseBytes = wt.replyInvalidSyntax()
				} else {
					responseBytes = wt.replyNumber(cache.Del(array[1].Str))
				}
			case "command":
				empty := make([]string, 0)
				responseBytes = wt.replyArray(empty)
			default:
				responseBytes = wt.replyCommandNotSupport(array[0].Str)
			}

		default:
			responseBytes = wt.replyInvalidSyntax()
		}
		appendAOFBuf(command, value)
		obj := &responseData{id: data.id, command: command, data: responseBytes}
		responseList.PushFront(obj)
	}

	return responseList
}

func handleClientsWithPendingWritesUsingThreads(responses *list.List) {
	fs := make([]func(), (*responses).Len())
	i := 0
	for e := responses.Front(); e != nil; e = e.Next() {
		obj := e.Value.(*responseData)
		fs[i] = func() {
			clientConn := clientsMap[obj.id]
			_, err := clientConn.wt.Write(obj.data)
			if strings.ToLower(obj.command) == "quit" {
				clientConn.conn.Close()
			}
			if err != nil {
				log.Error("response message error:", err)
			}
			clientConn.wt.Flush()
		}
		i = i + 1
	}
	ioPool.SyncRun(fs)
}
```


## AOF 持久化实现

### 实现原理方案

redis aof 持久化支持三种模式:always,everysec,no

| appendfsync 配置 | 解释                                                                                                           |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| always           | aof_buf 缓冲区所有内容写入并同步到 aof 文件                                                                    |
| everysec         | aof_buf 缓冲区所有内容写入 aof 文件，如果距离上一次同步文件超过 1s,则将同步 aof 文件，该操作由另外一个线程负责 |
| no               | aof_buf 缓冲区所有内容写入 aof 文件，但不对 aof 文件同步由系统决定何时同步                                     |

aof 保存的为操作 redis 的所有写操作，例如：set,sadd,incr 等。

aof 格式(redis aof 文件还包含版本信息，这里忽略)为：

```
*3
$3
set
$1
a
$1
a
*3
$3
set
$1
b
$1
b
```

aof 主要包含三个部分：

1. 持久化 aof 文件
2. 载入 aof 文件
3. 重写 aof 文件

### 实现方案

#### 持久化 aof 文件

实现思路：对客户端写事件，将该数据保存到 aof_buf 中，

定义一个全局切片`var aofBuf = make([]byte, 0)`

对于客户端发来的数据会读取为 Value,但是这个数据已经被处理，没有相应的符号和`\r\n`

在主处理逻辑中针对解析的 Value，循环遍历处理写事件

```
func appendAOFBuf(command string, value *Value) {
	if appendonly {
		switch strings.ToLower(command) {
		case "set", "del":
			raw := ValueToRow(value)
			aofBuf = append(aofBuf, raw...)
		}
	}
}
```

其中 ValueToRow 只是为了将结构化数据再转换为原始请求数据

```
func ValueToRow(value *Value) []byte {
	bs := []byte{value.Type}
	switch value.Type {

	case TypeSimpleError:
		str := []byte(strconv.Itoa(len(value.Str)) + CRLF + value.Str + CRLF)
		bs = append(bs, str...)
	case TypeSimpleString:
		log.Error("wait todo...")
	case TypeArray:
		array := value.Elems
		l := []byte(strconv.Itoa(len(array)) + CRLF)
		bs = append(bs, l...)
		for _, arg := range array {
			bs = append(bs, TypeBlobString)
			str := []byte(strconv.Itoa(len(arg.Str)) + CRLF + arg.Str + CRLF)
			bs = append(bs, str...)
		}
	default:
		log.Error("wait todo...")
	}
	return bs
}
```

接下来处理写 aof 文件，根据相应的 appendfsync 决定刷盘机制

```
func appendAOF(aofBuf []byte) {
	if n := len(aofBuf); n > 0 {
		aofHandle.Write(aofBuf)
		switch appendfsync {
		case "always":
			fsync()
		case "everysec":
			if nowTime := time.Now().UnixNano(); (nowTime - aofSyncTime) > 1e9 {
				go fsync()
			}
		}
	}

}
func fsync() {
	aofHandle.Flush()
	aofSyncTime = time.Now().UnixNano()
}
```

#### 载入 aof 文件

**redis 实现思路**：程序启动先检查是否存在 aof 文件，如果存在，则创建一个 fake client，读取 aof 文件，
然后利用伪客户端发送读取的命令。一直重复直至读取文件结束。

**本项目实现思路**：程序启动先检查是否存在 aof 文件，如果存在，读取 aof 文件，调用函数执行 set 操作。一直重复直至读取文件结束。

在解析文件的时候，需要计算相应的字节数，来确保下一次数据从哪里开始读取

```
func loadAofFile() {
	if aofHandle != nil {
		stat, err := aofHandle.file.Stat()
		if err != nil {
			log.Error("loadAofFile error:", err)
		}
		fileSize := stat.Size()

		if fileSize > 0 {
			i := int64(0)
			for i < fileSize {
				value, err := aofHandle.ReadValue(i)
				if err != nil {
					log.Error("loadAofFile ReadValue error:", err)
				}
				handleAOFReadCommand(value)
				i = i + value.Size
			}
		}

		log.Infof("loadAofFile success,aof size:%d", fileSize)
	}
}
```

```
func (handle *AOFHandle) ReadValue(offset int64) (*Value, error) {
	handle.file.Seek(offset, 1)
	value, error := handle.redisReader.ReadValue()
	return value, error
}
```

最后是根据解析的 Value 恢复数据

```
func handleAOFReadCommand(value *Value) {
	command := ""
	switch value.Type {
	case TypeSimpleError:
		log.Error(value.Err)
	case TypeSimpleString:
		log.Error("wait todo...")
	case TypeArray:
		array := value.Elems
		command = strings.ToLower(array[0].Str)
		switch command {
		case "set":
			cache.Set(array[1].Str, array[2].Str)
		case "del":
			cache.Del(array[1].Str)
		default:
			log.Error("wait todo...")
		}
	default:
		log.Error("wait todo...")
	}
}
```

#### 重写 aof 文件

1. 遍历 db,根据不同数据类型，转换成 resp 协议，生成的数据写成 temp 文件
2. 重写过程命令写入 aof 缓存区,aof 重写缓存区，待文件重写完成，原子替换原有 aof 文件
3. 将 aof 重写缓存区中的数据写入到新生成的 aof 文件中

为了避免堵塞服务器处理命令，重写过程会在子进程中执行。

## demo 测试

因为实现了 epoll，所以需要跨平台开发，这里我使用了 vs code 的 devcontainer 就可以实现在 window 平台上开发 linux 应用。用起来是真的爽歪歪，除了运行在 container，还可以运行在 wsl。不过我更倾向于 container。

client

```go
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:6379")
	if err != nil {
		log.Error(err)
		return
	}
	defer conn.Close()

	fmt.Println(send("set truman hello-world", conn))
	fmt.Println(send("get truman", conn))

}

func send(command string, conn net.Conn) string {
	conn.Write([]byte(command))
	buf := make([]byte, 1024)
	n, err1 := conn.Read(buf)
	if err1 != nil {
		log.Error(err1)
		return ""
	}

	return string(buf[:n])
}
```

除此以外，还可以用 docker container 运行 gocache 服务

## 说明

🙌 如果你阅读到这里，相信我们一定是**同道中人**，有任何想法，欢迎**私聊**我，微信号：**trumandu007**。
💡 如果你也是在西安地区从事 IT 相关工作，欢迎私信加入我建的 **『西安 IT 技术圈』**微信群,我们是一个什么样的群体？ [为什么要做『西安 IT 技术圈』](https://mp.weixin.qq.com/s?__biz=MzI4NTMwNTQ5Mg==&mid=2247483684&idx=1&sn=4c1f96c16463601a7e220a06649f4cd3)。
👬🏻 朋友，都看到这了，确定不关注一下么 👇

![](http://static.trumandu.top/trumandu_wechat_2022.png)
