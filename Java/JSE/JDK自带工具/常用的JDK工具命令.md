### JDK常用的工具--JDK8

| 工具名称  | 用途                        |
| --------- | --------------------------- |
| jps       | 列出已装载的JVM --- 常用    |
| jstack    | 打印线程堆栈信息 -- 常用    |
| jstat     | JVM监控统计信息 -- 常用     |
| jmap      | 打印JVM堆内对象情况 -- 常用 |
| jinfo     | 输出JVM配置信息-- 常用      |
| jconsole  | GUI监控工具                 |
| jvisualvm | GUI监控工具                 |
| jhat      | 堆离线分析工具              |
| jdb       | java进程调试工具            |
| jstatd    | 远程JVM监控统计信息         |

### jps命令

下面是我在自己Linux服务器上运行的(服务器上面跑了一个Tomcat)

```bash
[root@iZwz9jcjzd6wfh44nnnsv4Z ~]# jps
25057 Bootstrap
25116 Jps
```

### jstack命令

```bash
[root@iZwz9jcjzd6wfh44nnnsv4Z ~]# jstack --help
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
```

上面是使用的说明，下面来看一下实际的使用的打印情况

<https://www.cnblogs.com/chenpi/p/5377445.html>

