### Zookeeper集群的角色

**`Zookeeper`** 集群中的三种服务器角色：

- **Leader**

  进行投票的发起和决议，更新系统的状态

- **Follower**

  接收客户端的请求向客户端返回结果，在选举过程中参与投票

- **Observer**

  接收客户端连接，将写请求转发给leader节点。但Observer不参与投票，只同步leader的状态，Observer的目的是为了扩展系统，提高读取速度。

**`Zookeeper`** 集群中节点的四种状态：

- **LOOKING**: 寻找Leader的状态，在这种状态下集群没有Leader
- **FOLLOWING**: 表明当前服务器角色色Follower
- **LEADING**: 表明当前服务器角色是Leader
- **OBSERVING**:表明当前服务器角色是Observer

### Leader 选举过程

- #### 每一个Server都会发出一个投票

  初始状态下Server都会将自己作为 **`Leader`** 服务器来进行投票。每次投票包含的最基本的元素： myid 和 ZXID(事务号)。比如S1和S2两个服务节点。投票的为 S1 （1，0）, S2 (2, 0) 然后将投票的结果发给其他的机器

- #### 接收来自各个服务器的投票

  接收到投票后，会判断投票的有效性，包括检查是否是本轮投票和是否来自 **`LOOKING`** 状态的服务器

- #### 处理投票

  将自己的投票和其他服务器发来的投票进行比较，比较规则：

  - **优先检查ZXID, 大的服务器优先作为Leader**
  - **ZXID相同的，那么myid大的作为服务器的Leader**

  S1 服务器就会更新投票信息为 （2,0）。S2 不需要改。然后重新向集群中所有的机器发送出上一次投票信息即可。

- #### 统计投票

  服务器会统计所有的投票，超过半数以上的机器接收到相同的投票信息。这个时候就认为已经选出了Leader

- #### 改变服务器状态

  Leader 变更为 LEADING, Follower 变更为FOLLOWING状态

**Leader服务器会和每一个Follower/Observer服务器都建立TCP连接，同时为每个F/O都创建一个叫做LearnerHandler的实体。LearnerHandler主要负责Leader和F/O之间的网络通讯，包括数据同步，请求转发和Proposal提议的投票等。Leader服务器保存了所有F/O的LearnerHandler。**

### 服务运行期间的Leader投票

- #### 变更状态

  **非Observer服务器将状态全部变更为Looking，然后进入和初始化一样的选举流程。**