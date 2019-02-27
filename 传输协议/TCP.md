### 1. TCP简介

数据在 **`TCP`** 层称为 **`Stream`** ，数据分组称为分段（ **`Segment`** ）。作为比较，数据在 **`IP`** 层称为 **`Datagram`** ，数据分组称为分片（ **`Fragment`** ）。 **`UDP`**  中分组称为 **`Message`** 。

TCP数据结构：

![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/tcp%E6%95%B0%E6%8D%AE%E5%8C%85%E7%9A%84%E7%BB%93%E6%9E%84%E5%9B%BE.png?raw=true)

标志位的意思：

- NS — ECN-nonce
- CWR — Congestion Window Reduced。
- ECE — ECN-Echo有两种意思，取决于SYN标志的值。
- URG — 为1表示高优先级数据包，紧急指针字段有效。
- ACK — 为1表示确认号字段有效
- PSH — 为1表示是带有PUSH标志的数据，指示接收方应该尽快将这个报文段交给应用层而不用等待缓冲区装满(关闭连接的时候会看到这个字段)
- RST — 为1表示出现严重差错。可能需要重现创建TCP连接。还可以用于拒绝非法的报文段和拒绝连接请求。
- SYN — 为1表示这是连接请求或是连接接受请求，用于创建连接和使顺序号同步
- FIN — 为1表示发送方没有数据要传输了，要求释放连接。

### 2. TCP运作方式

TCP协议的运行可划分为三个阶段：连接创建(connection establishment)、数据传送（data transfer）和连接终止（connection termination）。操作系统将TCP连接抽象为套接字表示的本地端点（local end-point），作为编程接口给程序使用。在TCP连接的生命期内，本地端点要经历一系列的状态改变

#### 2.1 连接创建 --- TCP三次握手 

TCP用三路握手（或称三次握手，three-way handshake）过程创建一个连接。在连接创建过程中，很多参数要被初始化，例如序号被初始化以保证按序传输和连接的强壮性。

TCP连接的正常创建 一对终端同时初始化一个它们之间的连接是可能的。但通常是由一端打开一个套接字（socket）然后监听来自另一方的连接，这就是通常所指的被动打开（passive open）。服务器端被被动打开以后，用户端就能开始创建主动打开（active open）

三次握手的步骤：

1. 客户端通过向服务器发送一个 **`SYN`** 来创建一个主动打开，作为三路握手的一部分。客户端把这段连接的序号(seq)需要设定为一个随机数A(防止攻击) — 服务端处于 **`LISTEN`** 状态，客户端处于 **`SYN-SENT`** （同步发送状态）

   ![发送SYN图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E7%AC%AC%E4%B8%80%E6%AC%A1%E6%8F%A1%E6%89%8B.png?raw=true)

   从上图可以看出来 **seq=0** ， **flags** 的标记位显示的是 **SYN** 。这个测试是本地抓包。说明一下（130这个IP是客户端IP,184这个IP为服务端IP）。从图可以看出来第一次握手的 SYN 是从客户端发起。

2. 服务器端应当为一个合法的SYN回送一个SYN标志位和ACK标志位。ACK的确认码应为A+1,SYN/ACK包本身又有一个随机产生的序号B. — 服务处于 **`SYN-RCVD`** 同步接收状态

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E7%AC%AC%E4%BA%8C%E6%AC%A1%E6%8F%A1%E6%89%8B.png?raw=true)

   从上图可以看出这一次是由服务器发送的 **SYN/ACK** 的标记位。然后对应的返回给客户端的还有 **seq=0, Ack=1** 

3. 最后，客户端在发送一次ACK。当服务端收到这个ACK的时候就完成三路的握手。进入连接创建状态。此时包的序号被设定为收到的确认号A+1，而响应号则为B+1。 客户端和服务器都处于 **`ESTABE-LISHED`** (已建立的状态)

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E7%AC%AC%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B.png?raw=true)

   第三次是由客户端发送给服务器一个 ACK， seq=1 Ack=1。

   三次握手的图解如下：

![三次握手图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B%E5%9B%BE%E8%A7%A3.png?raw=true)

**但是通过抓包发现了一个问题按照TCP协议的说法seq生成应该是随机数，但是通过抓包发现每次建立连接的三次握手的第一步客户端发送SYN 的seq字段的值都是0？** 等待后续研究。

如果服务器接到了客户端发送SYN后回了 SYN-ACK 后客户端掉线了，服务端没有收到客户端回来的ACK,那么这个连接处于一个中间状态。既没成功也没失败。服务器端如果在一定时间内没有收到的TCP会重发SYN-ACK。会有一个重试的机制。在Linux下，默认重试五次。重试时间间隔从1秒开始翻倍，五次重试时间间隔1s,2s,4s,8s,16s。第五次发出后还要等32s才知道第五次超时了。所以总共需要63秒。

#### 2.2 终结通路 — TCP四次挥手

断开TCP连接可以是服务端也可以是客户端。我们以客户端为例子

1. 客户端先发送一个 **`FIN`** (表示发送方没有数据要传输了，要求释放连接)给服务器。

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B%E7%AC%AC1%E6%AC%A1.png?raw=true)

2. 服务器如果收到 **`FIN`** 给客户端回复一个 **`ACK`** 表示服务器已经收到。

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B%E7%AC%AC2%E6%AC%A1.png?raw=true)

3. 然后同时给客户端在发送一个 **`FIN`** 表示要求客户端释放连接

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B%E7%AC%AC3%E6%AC%A1.png?raw=true)

4. 如果客户端收到服务器的 **`FIN`** 消息，客户端再给服务器回复一个 **`ACK`** 表示消息收到了。这样就完成了一个四次挥手的动作

   ![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B%E7%AC%AC4%E6%AC%A1.png?raw=true)

四次挥手的图解(来源维基百科—懒的再去自己动手了)：

![图解](https://github.com/mxsm/document/blob/master/image/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/TCP_CLOSE.png?raw=true)