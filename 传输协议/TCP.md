### 1. TCP简介

数据在 **`TCP`** 层称为 **`Stream`** ，数据分组称为分段（ **`Segment`** ）。作为比较，数据在 **`IP`** 层称为 **`Datagram`** ，数据分组称为分片（ **`Fragment`** ）。 **`UDP`**  中分组称为 **`Message`**

### 2. TCP运作方式

TCP协议的运行可划分为三个阶段：连接创建(connection establishment)、数据传送（data transfer）和连接终止（connection termination）。操作系统将TCP连接抽象为套接字表示的本地端点（local end-point），作为编程接口给程序使用。在TCP连接的生命期内，本地端点要经历一系列的状态改变

#### 2.1 连接创建 --- TCP三次握手 

TCP用三路握手（或称三次握手，three-way handshake）过程创建一个连接。在连接创建过程中，很多参数要被初始化，例如序号被初始化以保证按序传输和连接的强壮性。

TCP连接的正常创建 一对终端同时初始化一个它们之间的连接是可能的。但通常是由一端打开一个套接字（socket）然后监听来自另一方的连接，这就是通常所指的被动打开（passive open）。服务器端被被动打开以后，用户端就能开始创建主动打开（active open）

三次握手的步骤：