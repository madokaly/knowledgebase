可以把 **TCP、UDP、HTTP** 放在同一张网络协议栈里理解。很多初学者容易把它们混在一起，实际上它们所处的层级不同：

```
应用层
┌──────────────────────────────┐
│ HTTP / HTTPS / DNS / FTP ... │
└──────────────────────────────┘
               ↓
传输层
┌──────────────┬───────────────┐
│     TCP      │      UDP      │
└──────────────┴───────────────┘
               ↓
网络层
┌──────────────────────────────┐
│             IP               │
└──────────────────────────────┘
               ↓
数据链路层 / 物理层
┌──────────────────────────────┐
│ Ethernet / Wi-Fi / 光纤 ... │
└──────────────────────────────┘
```

所以最核心的一句话是：

> **TCP / UDP 是传输层协议，负责“怎么把数据送过去”。**  
> **HTTP 是应用层协议，负责“数据按照什么格式和规则交流”。**

---

# 一、先理解整个网络通信过程

假设你在浏览器中访问：

```
https://www.example.com/index.html
```

一个极度简化的通信过程：

```
浏览器
  ↓
HTTP
构造请求：
GET /index.html HTTP/1.1

  ↓
TCP
保证 HTTP 数据可靠传输

  ↓
IP
负责寻找目标主机

  ↓
网卡 / Wi-Fi
真正发送电信号 / 无线信号
```

服务器收到后反过来拆：

```
网卡
 ↓
IP
 ↓
TCP
 ↓
HTTP
 ↓
Web服务器
```

可以理解成快递：

```
HTTP：
“包裹里面写什么、格式是什么”

TCP / UDP：
“这个包裹怎么运输”

IP：
“包裹送到哪个地址”

MAC：
“局域网里具体送到哪台设备”
```

---

# 二、TCP

TCP 全称：

```
Transmission Control Protocol
传输控制协议
```

TCP 最重要的特点：

```
面向连接
可靠传输
有序传输
字节流
全双工
流量控制
拥塞控制
```

可以简单理解：

> TCP 的设计目标不是“最快”，而是 **尽可能保证数据正确、完整、有序地到达对方。**

---

# 三、TCP 是面向连接的

TCP 在真正发送数据之前，需要先建立连接。

经典的：

```
TCP 三次握手
```

比如：

```
客户端                       服务器

   SYN
 ---------->
                           SYN + ACK
              <----------
   ACK
 ---------->

连接建立
```

## 第一次握手

客户端：

```
SYN = 1
Seq = x
```

意思：

> 我想和你建立连接，我的初始序列号是 x。

---

## 第二次握手

服务器：

```
SYN = 1
ACK = 1

Seq = y
Ack = x + 1
```

意思：

> 我收到你的连接请求了。  
> 我的初始序列号是 y。  
> 我期待你下一个数据序号是 x+1。

---

## 第三次握手

客户端：

```
ACK = 1

Seq = x + 1
Ack = y + 1
```

意思：

> 我也收到你的确认了。

然后：

```
TCP连接正式建立
```

---

# 四、为什么 TCP 是三次握手？

这是非常经典的面试题。

核心原因：

> **双方都需要确认自己和对方的发送、接收能力正常，并同步初始序列号。**

简单理解：

第一次：

```
Client -> Server

Client：
我能发送
```

第二次：

```
Server -> Client

Server：
我能接收
我也能发送
```

第三次：

```
Client -> Server

Client：
我能接收你的数据
```

最终双方确认：

```
Client：
发送 √
接收 √

Server：
发送 √
接收 √
```

---

# 五、TCP 为什么可靠？

TCP 的可靠性并不是“网络不会丢包”。

实际上：

```
IP 网络本身可能：

丢包
重复
乱序
延迟
```

TCP 是通过一系列机制，把一个“不可靠网络”包装成：

```
可靠字节流
```

主要依靠：

```
序列号 Sequence Number
确认应答 ACK
超时重传
快速重传
滑动窗口
流量控制
拥塞控制
校验和
```

---

# 六、TCP 序列号 Seq

假设发送：

```
HelloWorld
```

TCP 本质上把它看成字节：

```
H e l l o W o r l d
```

假设：

```
Seq = 100
```

发送了：

```
5字节
Hello
```

那么下一段可能是：

```
Seq = 105

World
```

也就是说：

```
Seq
```

表示：

> 当前 TCP 数据段中的 **第一个字节的位置**。

---

# 七、ACK 确认机制

例如：

客户端发送：

```
Seq = 100
长度 = 100字节
```

服务器收到后回复：

```
ACK = 200
```

意思不是：

```
我收到了第200字节
```

而是：

> **我已经收到 199 之前的数据，下一次希望收到 200。**

因此：

```
ACK = 下一段期望收到的序号
```

---

# 八、TCP 超时重传

假设：

```
Client

Seq=100
   --------X--------> Server
        丢包
```

Client 等了一段时间：

```
没有收到ACK
```

于是：

```
重新发送
```

```
Seq=100
   ----------------> Server
```

这就是：

```
Retransmission
重传
```

---

# 九、TCP 保证有序

网络可能出现：

```
发送：

1
2
3
```

实际到达：

```
1
3
2
```

TCP 不会直接：

```
1
3
2
```

交给应用程序。

因为 TCP 有序列号，所以可以重新排序：

```
1
2
3
```

再交给应用层。

因此：

> TCP 向应用程序提供的是 **有序字节流**。

---

# 十、TCP 是字节流协议

这一点对游戏开发尤其重要。

TCP 不保存：

```
消息边界
```

例如发送：

```
Send("ABC");

Send("DEF");
```

你不能保证接收端一定：

```
Recv -> "ABC"

Recv -> "DEF"
```

可能：

```
Recv -> "ABCDEF"
```

也可能：

```
Recv -> "A"
Recv -> "BCDE"
Recv -> "F"
```

因为 TCP 只关心：

```
连续字节流
```

而不关心：

```
你 Send 了几次
```

这就是 TCP 中著名的：

```
粘包 / 拆包问题
```

---

# 十一、TCP 粘包

例如发送三个消息：

```
[Hello]
[World]
[123]
```

接收可能变成：

```
HelloWorld123
```

这就是：

```
粘包
```

也可能：

```
Hel
loWorld
123
```

叫：

```
拆包
```

---

# 十二、如何解决 TCP 粘包？

常见方案：

## 方案一：固定长度

```
每条消息固定 100 Byte
```

例如：

```
[100字节]
[100字节]
[100字节]
```

缺点：

```
浪费空间
```

---

## 方案二：特殊分隔符

例如：

```
Hello\n
World\n
```

收到：

```
Hello\nWorld\n
```

根据：

```
\n
```

拆分。

HTTP Header 就使用了类似思想：

```
\r\n
```

---

## 方案三：长度 + 数据

游戏网络中非常常见：

```
┌─────────────┬──────────────┐
│ Length      │ Data         │
│ 4 bytes     │ N bytes      │
└─────────────┴──────────────┘
```

例如：

```
0005Hello
0005World
```

先读取：

```
Length
```

然后再读取指定字节。

---

# 十三、TCP 流量控制

TCP 有一个非常重要的问题：

假设：

```
发送方：
10GB/s

接收方：
1GB/s
```

如果疯狂发送：

```
接收方Buffer爆掉
```

所以 TCP 有：

```
Flow Control
流量控制
```

核心技术：

```
滑动窗口
Sliding Window
```

接收方告诉发送方：

```
我现在还能接收多少数据
```

这个值：

```
Receive Window
rwnd
```

例如：

```
Window = 64KB
```

意思：

> 你最多可以继续发送 64KB 数据。

---

# 十四、TCP 拥塞控制

流量控制解决：

```
发送方
    ↓
接收方处理不过来
```

拥塞控制解决的是：

```
整个网络处理不过来
```

例如：

```
Client
  ↓
Router
  ↓
Router
  ↓
Server
```

某个路由器：

```
带宽满了
```

开始大量丢包。

TCP 会判断：

```
网络拥塞
```

然后降低发送速度。

经典算法包括：

```
慢启动
Slow Start

拥塞避免
Congestion Avoidance

快速重传
Fast Retransmit

快速恢复
Fast Recovery
```

---

# 十五、TCP 四次挥手

TCP 关闭连接通常使用：

```
四次挥手
```

```
Client                           Server

 FIN
 -------------------------------->

                        ACK
 <--------------------------------

                        FIN
 <--------------------------------

 ACK
 -------------------------------->
```

为什么不是两次？

因为 TCP 是：

```
全双工
Full Duplex
```

也就是：

```
Client -> Server
```

和：

```
Server -> Client
```

是两个独立方向。

所以两个方向需要分别关闭。

---

# 十六、为什么三次握手、四次挥手？

经典面试问题。

建立连接时：

服务器可以：

```
ACK + SYN
```

一次一起发送。

所以：

```
3次
```

关闭连接时：

服务器收到：

```
FIN
```

只表示：

> 客户端不发送了。

但是服务器可能：

```
还有数据没发送完
```

所以先：

```
ACK
```

等数据发送结束后再：

```
FIN
```

因此通常：

```
4次
```

---

# 十七、TCP 常见应用

TCP 适合：

```
不能丢数据
需要顺序正确
可靠性重要
```

例如：

```
HTTP/1.1
HTTP/2

HTTPS

FTP

SSH

数据库连接

文件传输

登录系统

支付系统
```

游戏中也常用于：

```
登录
账号系统
商城
聊天
背包
任务
好友系统
```

---

# 十八、UDP

UDP 全称：

```
User Datagram Protocol
用户数据报协议
```

UDP 最大特点：

```
无连接
不保证可靠
不保证顺序
不自动重传
数据报协议
协议开销小
延迟低
```

可以粗略理解：

TCP：

```
“这个包你收到了吗？”

“没收到？”

“那我再发一次。”
```

UDP：

```
“给你。”

发完就不管了。
```

---

# 十九、UDP 不需要建立连接

TCP：

```
三次握手
↓
发送数据
```

UDP：

```
直接发送
```

因此 UDP：

```
启动延迟更低
协议逻辑更简单
```

---

# 二十、UDP 不保证可靠

例如：

```
Client

Packet 1
Packet 2
Packet 3
```

网络可能变成：

```
Server：

Packet 1
Packet 3
```

Packet 2：

```
丢了
```

UDP 默认：

```
不会重传
```

---

# 二十一、UDP 不保证顺序

发送：

```
1
2
3
```

可能收到：

```
3
1
2
```

UDP 不负责重新排序。

如果应用层需要顺序：

```
自己实现 Sequence Number
```

---

# 二十二、UDP 有消息边界

这是 TCP 和 UDP 非常大的区别。

UDP 是：

```
Datagram
数据报
```

发送：

```
Send("ABC")
Send("DEF")
```

接收时对应的是：

```
"ABC"

"DEF"
```

UDP 数据报本身有边界。

TCP：

```
ABCDEF
```

可能粘在一起。

所以：

```
TCP
字节流

UDP
数据报
```

这是非常高频的面试知识点。

---

# 二十三、UDP 为什么快？

经常有人说：

```
UDP比TCP快
```

更准确的说法应该是：

> UDP 的协议机制更简单，通常可以获得更低的协议开销和延迟。

因为它没有：

```
三次握手

ACK

自动重传

流量控制

TCP级别的拥塞控制

有序重组
```

UDP Header 也非常小。

---

# 二十四、UDP Header

UDP Header 只有：

```
8 Byte
```

结构：

```
┌────────────────────┐
│ Source Port        │ 2 Byte
├────────────────────┤
│ Destination Port   │ 2 Byte
├────────────────────┤
│ Length             │ 2 Byte
├────────────────────┤
│ Checksum           │ 2 Byte
└────────────────────┘
```

非常简单。

---

# 二十五、TCP Header

TCP Header 至少：

```
20 Byte
```

包含：

```
Source Port
Destination Port

Sequence Number

Acknowledgment Number

Flags
SYN
ACK
FIN
RST
PSH
URG

Window Size

Checksum

Options
```

因此 TCP 的功能明显更加复杂。

---

# 二十六、游戏为什么经常使用 UDP？

例如 FPS：

```
玩家位置
角色朝向
速度
移动输入
摄像机方向
```

假设：

```
t = 1.000

玩家位置：
(100, 0, 200)
```

这个数据包丢了。

100ms 后：

```
t = 1.100

玩家位置：
(130, 0, 250)
```

这时候旧数据：

```
(100,0,200)
```

已经没有太大意义。

如果 TCP 一定要：

```
重传旧位置
```

可能反而增加延迟。

因此：

```
实时性 > 完整性
```

的场景非常适合 UDP。

---

# 二十七、但游戏 UDP 不代表“不可靠”

这是非常重要的一点。

很多游戏：

```
UDP
+
应用层可靠协议
```

例如自己实现：

```
Sequence Number
ACK
重传
超时
排序
```

并且：

```
只让重要数据可靠
```

比如：

```
玩家位置
→ 不可靠

技能释放
→ 可靠

死亡
→ 可靠

拾取物品
→ 可靠

装备变化
→ 可靠
```

这样就可以获得：

```
TCP 的部分可靠性

+

UDP 的灵活低延迟
```

---

# 二十八、TCP vs UDP

最重要的一张表：

|特性|TCP|UDP|
|---|---|---|
|是否连接|面向连接|无连接|
|可靠性|可靠|默认不可靠|
|是否重传|是|否|
|顺序保证|是|否|
|数据形式|字节流|数据报|
|消息边界|没有|有|
|流量控制|有|无|
|拥塞控制|有|默认无|
|Header|≥20B|8B|
|延迟|相对高|相对低|
|实现复杂度|高|低|
|典型应用|HTTP、文件、登录|游戏、语音、直播、DNS|

---

# 二十九、HTTP

HTTP 全称：

```
HyperText Transfer Protocol

超文本传输协议
```

HTTP 属于：

```
应用层协议
```

它定义的是：

> 客户端和服务器 **按照什么格式交换数据**。

例如：

```
客户端：
我要 /index.html

服务器：
200 OK
这是文件内容
```

---

# 三十、HTTP 是请求-响应模型

HTTP 的核心：

```
Request
请求

Response
响应
```

流程：

```
Client

HTTP Request
      ↓
Server

HTTP Response
      ↓
Client
```

例如：

```
浏览器
   ↓
GET /index.html
   ↓
服务器
   ↓
200 OK
   ↓
HTML
```

---

# 三十一、HTTP Request

一个简单 HTTP 请求：

```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Chrome
Accept: text/html
```

结构：

```
请求行

Header

空行

Body
```

---

# 三十二、HTTP 请求行

例如：

```
GET /user/100 HTTP/1.1
```

包含：

```
GET
请求方法

/user/100
URL路径

HTTP/1.1
HTTP版本
```

---

# 三十三、HTTP 常见 Method

最常见：

```
GET
POST
PUT
DELETE
PATCH
HEAD
OPTIONS
```

---

## GET

获取资源：

```
GET /user/100
```

意思：

```
获取ID=100的用户
```

---

## POST

通常用于：

```
创建资源
提交数据
```

例如：

```
POST /login
```

Body：

```
{
    "username": "Tom",
    "password": "123456"
}
```

---

## PUT

通常表示：

```
整体更新资源
```

例如：

```
PUT /user/100
```

---

## PATCH

通常：

```
部分修改
```

例如只改：

```
nickname
```

---

## DELETE

```
DELETE /user/100
```

删除资源。

---

# 三十四、HTTP Header

HTTP Header 用来描述请求信息。

例如：

```
Host: www.example.com

Content-Type: application/json

Content-Length: 123

Authorization: Bearer xxx

Cookie: sessionId=123

User-Agent: Chrome

Accept: application/json
```

---

# 三十五、Content-Type

非常重要。

表示：

```
Body里面是什么数据
```

例如：

```
application/json
```

表示：

```
{
    "name": "Tom"
}
```

---

还有：

```
text/html

text/plain

image/png

application/octet-stream

multipart/form-data
```

其中上传文件经常使用：

```
multipart/form-data
```

---

# 三十六、HTTP Response

服务器返回：

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 26

{
    "name": "Tom"
}
```

结构：

```
状态行

Header

空行

Body
```

---

# 三十七、HTTP 状态码

HTTP 状态码非常重要。

总体：

```
1xx
信息

2xx
成功

3xx
重定向

4xx
客户端错误

5xx
服务器错误
```

---

# 三十八、最常见 HTTP 状态码

## 200 OK

```
请求成功
```

---

## 201 Created

```
资源创建成功
```

例如：

```
POST 创建用户
```

---

## 301

```
永久重定向
```

---

## 302

```
临时重定向
```

---

## 304

```
资源没有修改
```

常用于：

```
浏览器缓存
```

---

## 400 Bad Request

```
请求格式错误
```

---

## 401 Unauthorized

严格来说更接近：

```
没有完成身份认证
```

例如：

```
没登录
Token无效
```

---

## 403 Forbidden

```
服务器知道你是谁
但你没有权限
```

例如：

```
普通用户访问管理员接口
```

---

## 404 Not Found

```
资源不存在
```

---

## 500 Internal Server Error

```
服务器内部错误
```

---

## 502 Bad Gateway

常见于：

```
Nginx
↓
后端服务器
```

后端异常。

---

## 503 Service Unavailable

```
服务器暂时不可用
```

---

# 三十九、HTTP 本身是否可靠？

这里有一个非常常见的误区。

HTTP：

```
本身不负责可靠传输
```

传统 HTTP：

```
HTTP/1.1
HTTP/2
```

底层使用：

```
TCP
```

所以：

```
HTTP

↓

TCP
```

可靠性主要来自：

```
TCP
```

---

# 四十、HTTP 与 TCP 的关系

例如浏览器发送：

```
GET /index.html HTTP/1.1
```

HTTP 数据实际上会被放到：

```
TCP数据
```

中：

```
┌──────────────────────────────┐
│ IP Header                    │
├──────────────────────────────┤
│ TCP Header                   │
├──────────────────────────────┤
│ GET /index.html HTTP/1.1 ... │
└──────────────────────────────┘
```

所以：

```
HTTP = 数据规则

TCP = 数据运输方式
```

---

# 四十一、HTTP 1.0

HTTP/1.0 早期通常：

```
一个HTTP请求
建立一次TCP连接
```

例如：

```
TCP连接

GET html

关闭
```

然后图片：

```
重新建立TCP

GET image

关闭
```

这样效率很低。

---

# 四十二、HTTP/1.1 Keep-Alive

HTTP/1.1 默认可以：

```
复用TCP连接
```

例如：

```
TCP连接
    ↓
GET index.html
    ↓
GET a.png
    ↓
GET b.png
    ↓
GET style.css
```

不需要：

```
建立4次TCP
```

这就是：

```
Persistent Connection
持久连接
```

也经常称为：

```
Keep-Alive
```

---

# 四十三、HTTP/1.1 的队头阻塞

HTTP/1.1 通常存在：

```
Head-of-Line Blocking
队头阻塞
```

例如：

```
Request A
Request B
Request C
```

如果 A 很慢：

```
A
↓↓↓↓↓↓↓↓

B 等待

C 等待
```

这会影响性能。

---

# 四十四、HTTP/2

HTTP/2 最重要的升级之一：

```
多路复用
Multiplexing
```

一个 TCP 连接：

```
TCP
│
├─ Stream 1
│
├─ Stream 3
│
├─ Stream 5
│
└─ Stream 7
```

多个 HTTP 请求可以：

```
同时传输
```

不再像 HTTP/1.1 那样简单串行。

HTTP/2 还加入：

```
二进制分帧

Header压缩

多路复用
```

---

# 四十五、但 HTTP/2 仍然有 TCP 队头阻塞

虽然 HTTP/2 解决了：

```
HTTP层队头阻塞
```

但是底层：

```
还是TCP
```

TCP 要求：

```
数据必须按顺序交付
```

假设：

```
Packet 1
Packet 2  ← 丢了
Packet 3
Packet 4
```

即使：

```
3
4
```

到了，

TCP 可能仍然要等待：

```
2重传
```

之后再继续交给上层。

这就是：

```
TCP Head-of-Line Blocking
```

---

# 四十六、HTTP/3

HTTP/3 一个非常大的变化：

```
不再直接使用 TCP
```

而是：

```
HTTP/3
   ↓
QUIC
   ↓
UDP
```

也就是：

```
HTTP/1.1
    ↓
TCP

HTTP/2
    ↓
TCP

HTTP/3
    ↓
QUIC
    ↓
UDP
```

---

# 四十七、为什么 HTTP/3 使用 UDP？

不是因为：

```
HTTP突然不需要可靠性了
```

而是 QUIC 在 UDP 上：

```
自己实现可靠传输
```

包括：

```
ACK

重传

拥塞控制

流控制

多路复用

加密
```

可以理解为：

```
UDP

+

现代化可靠传输机制
```

---

# 四十八、QUIC 为什么能减少队头阻塞？

TCP：

```
一个连接
一个有序字节流
```

假设：

```
Stream A

一个包丢失
```

可能影响整个连接的数据交付。

QUIC：

```
Connection
│
├── Stream A
├── Stream B
└── Stream C
```

如果：

```
Stream A
```

发生丢包，

通常不会阻塞：

```
Stream B
Stream C
```

因此非常适合 HTTP/3。

---

# 四十九、HTTP 和 HTTPS

HTTPS：

```
HTTP Secure
```

传统 HTTPS：

```
HTTP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

而 HTTP：

```
HTTP
 ↓
TCP
 ↓
IP
```

TLS 负责：

```
加密

身份验证

数据完整性
```

---

# 五十、为什么 HTTPS 安全？

例如普通 HTTP：

```
username=Tom
password=123456
```

如果中间有人抓包：

```
可能直接看到
```

HTTPS 使用 TLS 加密后：

```
7a 23 f8 31 ...
```

中间人一般无法直接读懂内容。

HTTPS 主要解决：

```
窃听

数据篡改

服务器身份验证
```

---

# 五十一、HTTP 常用端口

默认：

```
HTTP
80

HTTPS
443
```

而 TCP / UDP 本身通过：

```
Port
端口
```

区分不同程序。

例如：

```
IP：
192.168.1.10
```

可能同时有：

```
80
Web服务

3306
MySQL

22
SSH

7777
游戏服务器
```

---

# 五十二、IP 和 Port 的关系

可以理解：

```
IP
=
哪一栋楼

Port
=
哪一个房间
```

例如：

```
192.168.1.100:8080
```

表示：

```
192.168.1.100
这台计算机

8080
这个程序 / 服务端口
```

实际上网络连接通常可以用：

```
Source IP
Source Port

Destination IP
Destination Port
```

来描述。

例如：

```
192.168.1.100:50321

->

8.8.8.8:443
```

---

# 五十三、Socket 又是什么？

在写 C++ / Unity 网络程序时一定会遇到：

```
Socket
套接字
```

Socket 不是：

```
TCP
```

也不是：

```
UDP
```

Socket 可以理解成：

> 操作系统提供给应用程序使用网络协议的编程接口。

例如：

```
TCP Socket
```

```
Socket socket =
    new Socket(
        AddressFamily.InterNetwork,
        SocketType.Stream,
        ProtocolType.Tcp);
```

其中：

```
SocketType.Stream
```

表示：

```
字节流
```

```
ProtocolType.Tcp
```

表示：

```
TCP
```

UDP：

```
Socket socket =
    new Socket(
        AddressFamily.InterNetwork,
        SocketType.Dgram,
        ProtocolType.Udp);
```

其中：

```
Dgram
=
Datagram
数据报
```

---

# 五十四、TCP Socket 通信模型

服务器：

```
Socket()

↓

Bind()

↓

Listen()

↓

Accept()

↓

Recv()
Send()

↓

Close()
```

客户端：

```
Socket()

↓

Connect()

↓

Send()
Recv()

↓

Close()
```

对应：

```
Client                     Server

Socket                      Socket
                             |
                             Bind
                             |
                             Listen
                             |
Connect -----------------> Accept

Send --------------------> Recv

Recv <-------------------- Send
```

---

# 五十五、UDP Socket 通信模型

UDP 没有三次握手，所以更简单：

```
Client                      Server

Socket                       Socket
                              |
                              Bind

SendTo -------------------> RecvFrom

RecvFrom <---------------- SendTo
```

通常使用：

```
SendTo
RecvFrom
```

因为 UDP 每一个数据报都需要知道：

```
发给谁
从谁收到
```

---

# 五十六、TCP / UDP / HTTP 一张图彻底理解

可以把整个关系理解成：

```
Application
应用程序

浏览器
游戏
Unity Client
游戏服务器
          │
          │
          ▼
┌───────────────────────────────┐
│ 应用层                         │
│                               │
│ HTTP / HTTPS / WebSocket      │
│ DNS / FTP / 自定义游戏协议      │
└───────────────────────────────┘
          │
          ▼
┌───────────────────────────────┐
│ 传输层                         │
│                               │
│ TCP             UDP           │
│                               │
│ 可靠             快、灵活      │
│ 字节流           数据报         │
└───────────────────────────────┘
          │
          ▼
┌───────────────────────────────┐
│ 网络层                         │
│                               │
│ IP                            │
│                               │
│ 负责寻址 / 路由                 │
└───────────────────────────────┘
          │
          ▼
┌───────────────────────────────┐
│ 数据链路层                     │
│                               │
│ Ethernet / Wi-Fi / MAC        │
└───────────────────────────────┘
```

---

# 五十七、用游戏开发场景理解三者

假设你开发一个 UE / Unity 网络游戏。

## 登录

```
账号
密码
Token
角色信息
```

数据：

```
不能丢
```

可以使用：

```
HTTP / HTTPS
```

底层：

```
TCP
```

例如：

```
Client

POST /login

↓

HTTP

↓

TCP

↓

Login Server
```

---

## 游戏实时移动

例如：

```
Position

Rotation

Velocity
```

这种数据：

```
非常频繁

旧数据价值低

更看重实时性
```

通常更适合：

```
UDP
```

---

## 商城购买

```
购买商品
支付
扣金币
获得物品
```

要求：

```
绝对不能随便丢
```

通常：

```
TCP
HTTP / HTTPS
可靠 RPC
```

---

# 五十八、一个非常重要的误区

不能简单理解成：

```
TCP = 安全
UDP = 不安全
```

“可靠”和“安全”不是一个概念。

TCP：

```
可靠性
```

主要指：

```
不丢
有序
重传
```

但 TCP 默认：

```
不加密
```

别人仍然可能抓包看到内容。

真正的加密通常：

```
TLS
```

例如：

```
HTTPS

=

HTTP + TLS
```

---

# 五十九、另一个重要误区

也不能说：

```
UDP一定比TCP快
```

更准确：

```
UDP机制更少
应用可以更灵活地控制延迟
```

但实际速度受到：

```
网络带宽
RTT
丢包率
拥塞
应用协议
服务器
```

等大量因素影响。

像 QUIC：

```
虽然使用UDP
```

里面实际上也实现了：

```
ACK
重传
拥塞控制
流量控制
```

---

# 六十、面试最常问的 TCP / UDP / HTTP 问题

建议重点掌握这些：

```
1. TCP和UDP有什么区别？

2. 为什么TCP可靠？

3. TCP三次握手过程？

4. 为什么需要三次握手？

5. TCP四次挥手过程？

6. 为什么挥手通常是四次？

7. TCP为什么会粘包？

8. TCP如何解决粘包？

9. UDP有没有粘包？

10. TCP滑动窗口是什么？

11. TCP流量控制是什么？

12. TCP拥塞控制是什么？

13. TCP为什么需要序列号？

14. ACK是什么意思？

15. 超时重传是什么？

16. TCP和HTTP是什么关系？

17. HTTP和HTTPS区别？

18. GET和POST区别？

19. 常见HTTP状态码？

20. HTTP/1.1和HTTP/2区别？

21. HTTP/2和HTTP/3区别？

22. QUIC为什么基于UDP？

23. Socket是什么？

24. IP和Port有什么区别？
```

如果是**游戏客户端 / UE / Unity 网络方向面试**，还建议重点补：

```
TCP粘包/拆包
UDP可靠传输
可靠UDP
序列号
ACK
RTT
丢包
延迟
抖动
预测
插值
快照同步
帧同步
状态同步
RPC
```

其中有一条主线特别值得记住：

```
TCP
 │
 ├─ 面向连接
 ├─ Seq / ACK
 ├─ 重传
 ├─ 有序
 ├─ 字节流
 ├─ 滑动窗口
 └─ 拥塞控制


UDP
 │
 ├─ 无连接
 ├─ 数据报
 ├─ 无默认重传
 ├─ 无默认顺序保证
 └─ 低开销 / 高灵活性


HTTP
 │
 ├─ 应用层
 ├─ Request / Response
 ├─ Method
 ├─ Header
 ├─ Body
 └─ Status Code
        │
        ├─ HTTP/1.1 → TCP
        ├─ HTTP/2   → TCP
        └─ HTTP/3   → QUIC → UDP
```

这张关系图如果真正理解了，后面再学习 **Socket、WebSocket、HTTPS、QUIC、游戏网络同步、可靠 UDP、帧同步和状态同步**都会顺很多。