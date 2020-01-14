### 1. Raft心跳维持

在选取出来leader后，raft通过心跳来维持leader的存在。通过不断的给follwer发送心跳来告诉follwer还存在，如果在一段时间内follwer没有收到心跳，那么follwer就会发起选leader的过程。
### 2. nacos(1.1.4版本)源码分析
在前面的Nacos源码解析之Raft一致性算法选leader分析过了在类 ***RaftCore*** 类中注册了一个 ***HeartBeat*** 的定时任务，来定时发送心跳。首先看一下 **GlobalExecutor#registerHeartbeat** 方法

```java
   //public static final long TICK_PERIOD_MS = TimeUnit.MILLISECONDS.toMillis(500L);
    public static void registerHeartbeat(Runnable runnable) {
        executorService.scheduleWithFixedDelay(runnable, 0, TICK_PERIOD_MS, TimeUnit.MILLISECONDS);
    }
```
通过代码可以看到这个定时任务每500毫秒执行一次。然后在看一下 ***HeartBeat*** 的实现：

```java
 public class HeartBeat implements Runnable {
      @Override
        public void run() {
           try {

                if (!peers.isReady()) {
                    return;
                }

                RaftPeer local = peers.local();
                local.heartbeatDueMs -= GlobalExecutor.TICK_PERIOD_MS;
                //到期时间的判定
                if (local.heartbeatDueMs > 0) {
                    return;
                }

                local.resetHeartbeatDue();

                sendBeat();
            } catch (Exception e) {
                Loggers.RAFT.warn("[RAFT] error while sending beat {}", e);
            }
        }
     public void sendBeat() throws IOException, InterruptedException {
         
         //省略代码下面单独分析
     }    
 }
```
上面的run方法的代码可以看出来获取当前机器节点的 **RaftPeer** 然后判断心跳到期时间来判断是否执行接下来的 **sendBeat()** 方法：

```java
public void sendBeat() throws IOException, InterruptedException {
            RaftPeer local = peers.local();
            //非leader或者单机模式不发送
            if (local.state != RaftPeer.State.LEADER && !STANDALONE_MODE) {
                return;
            }

            //重置leader的任期时间--任期延续
            local.resetLeaderDue();

            // build data
            JSONObject packet = new JSONObject();
            packet.put("peer", local);

            JSONArray array = new JSONArray();

            if (switchDomain.isSendBeatOnly()) {
                //省略日志打印
            }

            if (!switchDomain.isSendBeatOnly()) {
                for (Datum datum : datums.values()) {

                    JSONObject element = new JSONObject();

                    if (KeyBuilder.matchServiceMetaKey(datum.key)) {
                        element.put("key", KeyBuilder.briefServiceMetaKey(datum.key));
                    } else if (KeyBuilder.matchInstanceListKey(datum.key)) {
                        element.put("key", KeyBuilder.briefInstanceListkey(datum.key));
                    }
                    element.put("timestamp", datum.timestamp);

                    array.add(element);
                }
            }
            packet.put("datums", array);
            // broadcast
            Map<String, String> params = new HashMap<String, String>(1);
            params.put("beat", JSON.toJSONString(packet));

            String content = JSON.toJSONString(params);

            ByteArrayOutputStream out = new ByteArrayOutputStream();
            GZIPOutputStream gzip = new GZIPOutputStream(out);
            gzip.write(content.getBytes(StandardCharsets.UTF_8));
            gzip.close();

            byte[] compressedBytes = out.toByteArray();
            String compressedContent = new String(compressedBytes, StandardCharsets.UTF_8);

             for (final String server : peers.allServersWithoutMySelf()) {
                try {
                    final String url = buildURL(server, API_BEAT);
 
                    HttpClient.asyncHttpPostLarge(url, null, compressedBytes, new AsyncCompletionHandler<Integer>() {
                        @Override
                        public Integer onCompleted(Response response) throws Exception {
                            if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                                MetricsMonitor.getLeaderSendBeatFailedException().increment();
                                return 1;
                            }
                            //更新状态
                            peers.update(JSON.parseObject(response.getResponseBody(), RaftPeer.class));
                            return 0;
                        }

                        @Override
                        public void onThrowable(Throwable t) {
                          MetricsMonitor.getLeaderSendBeatFailedException().increment();
                        }
                    });
                } catch (Exception e) {
                    MetricsMonitor.getLeaderSendBeatFailedException().increment();
                }
            }
        }
```
通过上面可以一步发送来更新状态。看一下 **RaftController#beat** 方法主要是处理心跳：

```java
    @NeedAuth
    @PostMapping("/beat")
    public JSONObject beat(HttpServletRequest request, HttpServletResponse response) throws Exception {

        String entity = new String(IoUtils.tryDecompress(request.getInputStream()), StandardCharsets.UTF_8);
        String value = URLDecoder.decode(entity, "UTF-8");
        value = URLDecoder.decode(value, "UTF-8");

        JSONObject json = JSON.parseObject(value);
        JSONObject beat = JSON.parseObject(json.getString("beat"));
        
        //心跳处理
        RaftPeer peer = raftCore.receivedBeat(beat);

        return JSON.parseObject(JSON.toJSONString(peer));
    }
```
心跳处理主要是通过 ***RaftCore#receivedBeat*** 方法处理：

```java
 final RaftPeer local = peers.local();

        //解析远程的RaftPeer
        final RaftPeer remote = new RaftPeer();
        remote.ip = beat.getJSONObject("peer").getString("ip");
        remote.state = RaftPeer.State.valueOf(beat.getJSONObject("peer").getString("state"));
        remote.term.set(beat.getJSONObject("peer").getLongValue("term"));
        remote.heartbeatDueMs = beat.getJSONObject("peer").getLongValue("heartbeatDueMs");
        remote.leaderDueMs = beat.getJSONObject("peer").getLongValue("leaderDueMs");
        remote.voteFor = beat.getJSONObject("peer").getString("voteFor");

        //remote RaftPeer状态判断
        if (remote.state != RaftPeer.State.LEADER) {
            Loggers.RAFT.info("[RAFT] invalid state from master, state: {}, remote peer: {}",
                remote.state, JSON.toJSONString(remote));
            throw new IllegalArgumentException("invalid state from master, state: " + remote.state);
        }

        if (local.term.get() > remote.term.get()) {
            Loggers.RAFT.info("[RAFT] out of date beat, beat-from-term: {}, beat-to-term: {}, remote peer: {}, and leaderDueMs: {}"
                , remote.term.get(), local.term.get(), JSON.toJSONString(remote), local.leaderDueMs);
            throw new IllegalArgumentException("out of date beat, beat-from-term: " + remote.term.get()
                + ", beat-to-term: " + local.term.get());
        }

        if (local.state != RaftPeer.State.FOLLOWER) {

            Loggers.RAFT.info("[RAFT] make remote as leader, remote peer: {}", JSON.toJSONString(remote));
            // mk follower
            local.state = RaftPeer.State.FOLLOWER;
            local.voteFor = remote.ip;
        }

        final JSONArray beatDatums = beat.getJSONArray("datums");
        //重置本地的心跳发送时间和leader的任期时间
        local.resetLeaderDue();
        local.resetHeartbeatDue();

        peers.makeLeader(remote);
        //省略部分代码
}
```
获取Leader发送的心跳信息然后进行判断，最后更新本地的RaftPeer信息