### 1.  JVM的内存模型

从网上找了几个内存的模型和自己画的一个

![图片](https://github.com/mxsm/document/blob/master/image/JSE/JVMmodel.png?raw=true)

![图片](https://github.com/mxsm/document/blob/master/image/JSE/JVM%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E5%9B%BE%E8%A7%A3.png?raw=true)

![图片](https://github.com/mxsm/document/blob/master/image/JSE/%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E5%9B%BE%E8%A7%A3.png?raw=true)

**线程共享：**

- **方法区：** 

  在 **`Java`** 虚拟机中，方法区是可供各线程共享的运行时内存区域。在HotSpot中，设计者将方法区纳入GC分代收集。HotSpot虚拟机堆内存被分为新生代和老年代，对堆内存进行分代管理，所以HotSpot虚拟机使用者更愿意将方法区称为老年代。

- **堆：**

  堆内存是 **`JVM`** 所有线程共享的部分，在虚拟机启动的时候就已经创建了。所有的对象和数组都在堆上进行分配。这部分空间可以通过GC进行垃圾回收的。当申请不到空间时会抛出 **`OutOfMemoryError`** 。

**线程隔离：** 

- **PC寄存器**

  PC寄存器也叫程序计数器，可以看成当前线程所执行的字节码的行号指示器。在任何一个确定的时刻，一个处理器都只会执行一条线程中的指令。应酬为了线程切换能够正确的恢复到正确的执行位置。每条线程都需要一个独立的程序计数器，PC寄存器所以是线程私有的和特定的线程绑定在一起的。**执行的是Java方法该寄存器保存的是当前指令的地址。如果执行的是本地方法PC寄存器就是空的。 ** 

- **本地方法栈**

  保存native方法进入区域的地址

- **虚拟机栈**

  