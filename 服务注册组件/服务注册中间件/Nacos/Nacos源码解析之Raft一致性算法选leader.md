### 1. Raft一致性算法选主

nacos项目的分布式一致性是通过Raft来实现，下面来看一下Raft算法在nacos中的实现。nacos项目是一个SpringBoot项目。Raft算法的实现在 ***naming*** 模块中。
### 2. 实现源码分析
选leader是由 ***RaftController*** 的 ***vote*** 方法来实现。看一下源码：

```java
    @PostMapping("/vote")
    public JSONObject vote(HttpServletRequest request, HttpServletResponse response) throws Exception {
        
        //接收处理投票
        RaftPeer peer = raftCore.receivedVote(
            JSON.parseObject(WebUtils.required(request, "vote"), RaftPeer.class));

        return JSON.parseObject(JSON.toJSONString(peer));
    }
```
通过上面的源码可以看出来有两个重要的类：  
- **RaftCore**  
  Raft算法实现的核心类，包含了Raft的实现的算法逻辑
- **RaftPeer**  
  选leader发送Body的类

下面来看一下主要的接收到投票的处理逻辑 **RaftCore#receivedVote** ：

```java
public synchronized RaftPeer receivedVote(RaftPeer remote) {
        //判断RaftPeerSet中是否包含RaftPeer
        if (!peers.contains(remote)) {
            throw new IllegalStateException("can not find peer: " + remote.ip);
        }
        
        //获取当前的RaftPeer
        RaftPeer local = peers.get(NetUtils.localServer());
        
        //本地的RaftPeer任期(term) >= 远端的
        if (remote.term.get() <= local.term.get()) {
            //省略部分代码
            if (StringUtils.isEmpty(local.voteFor)) {
                local.voteFor = local.ip;
            }
            //这样投票给到当前本地的RaftPeer
            return local;
        }
        //否则投票给远程的RaftPeer
        local.resetLeaderDue();
        //本地的RaftPeer state 改为FOLLOWER
        local.state = RaftPeer.State.FOLLOWER;
        local.voteFor = remote.ip;
        local.term.set(remote.term.get());

        return local;
}
```
通过上面的逻辑代码分析，通过远端发送的投票信息的 ***RaftPeer*** 通过任期和本地任期做对比，如果本地任期大于等于远端的RaftPeer的任期，那么投票就是给到本地。  
那又是怎么发送这些投票信息的。这就需要看一下 ***RaftCore*** 类的源码接下来分析，首先看一下初始化方法 ***RaftCore#init*** ：

```java
 private ScheduledExecutorService executor = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);

            t.setDaemon(true);
            t.setName("com.alibaba.nacos.naming.raft.notifier");

            return t;
        }
    });


    @PostConstruct
    public void init() throws Exception {

       

        executor.submit(notifier);

        long start = System.currentTimeMillis();

        raftStore.loadDatums(notifier, datums);

        setTerm(NumberUtils.toLong(raftStore.loadMeta().getProperty("term"), 0L));

        Loggers.RAFT.info("cache loaded, datum count: {}, current term: {}", datums.size(), peers.getTerm());

        while (true) {
            if (notifier.tasks.size() <= 0) {
                break;
            }
            Thread.sleep(1000L);
        }

        initialized = true;

        Loggers.RAFT.info("finish to load data from disk, cost: {} ms.", (System.currentTimeMillis() - start));
        //注册选master的线程
        GlobalExecutor.registerMasterElection(new MasterElection());
         //注册发送心跳的线程
        GlobalExecutor.registerHeartbeat(new HeartBeat());

        Loggers.RAFT.info("timer started: leader timeout ms: {}, heart-beat timeout ms: {}",
            GlobalExecutor.LEADER_TIMEOUT_MS, GlobalExecutor.HEARTBEAT_INTERVAL_MS);
    }
```
主要是通过  ***GlobalExecutor.registerMasterElection(new MasterElection());*** 和  ***GlobalExecutor.registerHeartbeat(new HeartBeat());*** 一个是选master的运行逻辑，一个是发送心跳的逻辑(另外文章分析)。

```java
    //public static final long TICK_PERIOD_MS = TimeUnit.MILLISECONDS.toMillis(500L);
    public static void registerMasterElection(Runnable runnable) {
        executorService.scheduleAtFixedRate(runnable, 0, TICK_PERIOD_MS, TimeUnit.MILLISECONDS);
    }
```
通过定时已固定500毫秒执行一次 ***MasterElection*** 。下面来看一下 ***MasterElection*** 的源码：

```java
public class MasterElection implements Runnable {
    //省略代码
}
```
通过实现Runnable来进行，里面主要有两个方法一个是 **run** 和 **sendVote** 。下面来看一下这两个方法的源码：

```
        @Override
        public void run() {
            try {
                
                if (!peers.isReady()) {
                    return;
                }
                //获取当前系统的RaftPeer
                RaftPeer local = peers.local();
                local.leaderDueMs -= GlobalExecutor.TICK_PERIOD_MS;
                
                //follower就是根据这个变量判断是否要重新选leader的
                if (local.leaderDueMs > 0) {
                    return;
                }

                // 重置到期时间
                local.resetLeaderDue();
                local.resetHeartbeatDue();
                //发送投票
                sendVote();
            } catch (Exception e) {
                Loggers.RAFT.warn("[RAFT] error while master election {}", e);
            }

        }
```
***run*** 方法中主要做了三个事情：
1. 减少到期时间然后判断是否大于0
2. 重置Leader任期到期时间和心跳到期时间
3. 发送投票调用投票的接口(/v1/ns/raft/vote)

上面有一段代码： **local.leaderDueMs > 0** 这个来判断follower是否需要进行选举。在刚刚启动的系统的时候所有的节点状态都是follwer状态。下面来看一下投票发送的源码实现：

```java
//下面代码省略了部分日志打印的代码
public void sendVote() {
            //获取本地RaftPeer
            RaftPeer local = peers.get(NetUtils.localServer());
            //重置RaftPeerSet
            peers.reset();

            local.term.incrementAndGet();
            local.voteFor = local.ip; //投票给自己
            //设置当前RaftPeer状态为CANDIDATE 也就是Raft算法中提到的状态
            local.state = RaftPeer.State.CANDIDATE;

            Map<String, String> params = new HashMap<>(1);
            params.put("vote", JSON.toJSONString(local));
            //给集群中除了自身以外的集群节点发送投票请求
            for (final String server : peers.allServersWithoutMySelf()) {
                final String url = buildURL(server, API_VOTE);
                try {
                    //异步调用处理投票结果
                    HttpClient.asyncHttpPost(url, null, params, new AsyncCompletionHandler<Integer>() {
                        @Override
                        public Integer onCompleted(Response response) throws Exception {
                            if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                                return 1;
                            }

                            RaftPeer peer = JSON.parseObject(response.getResponseBody(), RaftPeer.class);
                            //处理投票的结果
                            peers.decideLeader(peer);

                            return 0;
                        }
                    });
                } catch (Exception e) {
                    Loggers.RAFT.warn("error while sending vote to server: {}", server);
                }
            }
        }
```
发送投票的大概逻辑就是：
1. 获取本地的RaftPeer并且重置RaftPeerSet。
2. 本地的RaftPeer投票给自己，RaftPeer.voteFor=当前服务器的IP
3. 获取除了自身以外的集群节点的服务的IP地址异步发送投票消息
4. 处理返回的投票结果。

在 **MasterElection#sendVote** 处理包含了前面三步的处理。第4步的处理在  ***`RaftPeerSet#decideLeader`*** 中进行了处理。接下来分析如何决定谁是Leader:

```java
    public RaftPeer decideLeader(RaftPeer candidate) {
        peers.put(candidate.ip, candidate);

        SortedBag ips = new TreeBag();
        int maxApproveCount = 0;
        String maxApprovePeer = null;
        for (RaftPeer peer : peers.values()) {
            if (StringUtils.isEmpty(peer.voteFor)) {
                continue;
            }

            ips.add(peer.voteFor);
            if (ips.getCount(peer.voteFor) > maxApproveCount) {
                maxApproveCount = ips.getCount(peer.voteFor);
                maxApprovePeer = peer.voteFor;
            }
        }
        //判断投票数是否大于集群数量的一半
        if (maxApproveCount >= majorityCount()) {
            RaftPeer peer = peers.get(maxApprovePeer);
            peer.state = RaftPeer.State.LEADER;

            if (!Objects.equals(leader, peer)) {
                leader = peer;
                applicationContext.publishEvent(new LeaderElectFinishedEvent(this, leader));
                Loggers.RAFT.info("{} has become the LEADER", leader.ip);
            }
        }

        return leader;
    }
```
上面代码主要分析了选leader的这个过程。
