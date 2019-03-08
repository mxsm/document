### ByteBuf --- netty的数据容器

网络通信涉及到字节序列的移动。高效易用的数据结构必不可少。

#### ByteBuf如何工作

**`ByteBuf`** 的布局结构和状态图：

![图解](https://github.com/mxsm/document/blob/master/image/netty/bytebuf%E7%BB%93%E6%9E%84%E5%9B%BE.png?raw=true)

**`ByteBuf`** **维护了两个不同的索引**:

- **readerIndex：**用于读取
- **writerIndex：** 用于写入