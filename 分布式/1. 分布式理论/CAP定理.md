## 1. 什么是 CAP？ ##
CAP定理指的是在一个分布式系统中， Consistency（一致性）、 Availability（可用性）、  
Partition tolerance（分区容错性），三者不可兼得。CAP模型图如下：  

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/distributed/cap-model.jpg)

如上图，如果是最多同时满足两项，那我们可以有三个组合：CA、CP、AP。在聊这三个组合之前，  
我们先分别看一下 Consisteny（一致性）、Availability（可用性）、Partition tolerance（分区容错性）的含义。

### 1.1 Partition tolerance（分区容错性） ###
大多数分布式系统都分布在多个子网络。每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。当节点之间的网络通信出现问题之后，仍然能够对外提供满足一致性和可用性的服务，除非整个网络环境都发生了故障。
### 1.2 Consistency（一致性）###
指数据在多个副本之间能够保持一致的特性（严格的一致性）。简单来说就是在进行写操作的时候，只有当所有副本上的数据都同步成功之后才返回，可以理解为主从节点的同步复制，非一致性可以理解为主从节点的异步复制，异步复制可能导致主从节点上数据不一致。
### 1.3 Availability（可用性）###
指系统提供的服务必须一直处于可用的状态，每次请求都能获取到非错的响应（不保证获取的数据为最新数据）。  
以下用图来说明：   

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/distributed/cap-framework.jpg)

一致性的要求是指，对于任何客户端（上图Actor）来说，每次的读操作，都能获得最新的数据。即，当有客户端向A节点写入了新数据之后，其它客户端从B节点中进行读操作所获得的数据必须也是最新的，是与A节点数据保持一致的。  

可用性的要求是指，每个请求都能在合理的时间内获得符合预期的响应（不保证获取的结果是最新的数据）。
按照上图来看就是，客户端只要向A节点或B节点发起请求后，只要这两个节点收到了请求，就必须响应给客户端，但不需要保证响应的值是否正确。

## 2. CAP 怎么应用？ ##
虽然我们知道有 CA、CP、AP 三种组合方式，但是在分布式系统的结构下，网络是不可能做到100%可靠的。既然网络不能保证绝对可靠，那 P（分区容错性）就是一个必选项了。原因如下：  

如果选择 CA组合，放弃 P（分区容错性）。还是以最上面的图中A和B节点来举例，当发生节点间网络故障时，为了保证 C（一致性），那么就必须将系统锁住，不允许任何写入操作，否者就会出现节点之间数据不一致了。但是锁住了系统，就意味着当有写请求进来的时候，系统是不可用的，这一点又违背了 A（可用性）原则。 

因此分布式系统理论上是不可能有CA组合的，所以我们只能选择 CP 和 AP组合架构。  
下面我们来详细看一下 CP架构 和 AP架构的特点：

### 2.1 CP架构 ###
如下图，由于网络问题，节点A和节点B之前不能互相通讯。当有客户端（上图Actor）向节点A进行写入请求时（准备写入Message 2），节点A会不接收写入操作，导致写入失败，这样就保证了节点A和节点B的数据一致性，即保证了Consisteny（一致性）。  

然后，如果有另一个客户端（上图另一个Actor）向B节点进行读请求的时候，B请求返回的是网络故障之前所保存的信息（Message 1），并且这个信息是与节点A一致的，是整个系统最后一次成功写入的信息，是能正常提供服务的，即保证了Partition tolerance（分区容错性）。  

上述情况就是保障了CP架构，但放弃了Availability（可用性）的方案。

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/distributed/cp-framework.jpg)

### 2.2 AP架构 ###
如下图，由于网络问题，节点A和节点B之前不能互相通讯。当有客户端（上图Actor）向节点A进行写入请求时（准备写入Message 2），节点A允许写入，请求操作成功。但此时，由于A和B节点之前无法通讯，所以B节点的数据还是旧的（Message 1）。当有客户端向B节点发起读请求时候，读到的数据是旧数据，与在A节点读到的数据不一致。但由于系统能照常提供服务，所以满足了Availability（可用性）要求。  

因此，这种情况下，就是保障了AP架构，但其放弃了 Consisteny（一致性）。

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/distributed/ap-framework.jpg)


## 3. 使用注意事项 ##
了解了CAP定理后，对于开发者而言，当我们构建服务的时候，就需要根据业务特性作出权衡考虑，哪些点是当前系统可以取舍的，哪些是应该重点保障的。

对于多数大型互联网应用的场景，主机众多、部署分散。而且现在的集群规模越来越大，所以节点故障、网络故障是常态。这种应用一般要保证服务可用性达到N个9，即保证P和A，只有舍弃C（退而求其次保证最终一致性）。虽然某些地方会影响客户体验，但没达到造成用户流程的严重程度。

对于涉及到钱财这样不能有一丝让步的场景，C必须保证。网络发生故障宁可停止服务，这是保证CA，舍弃P。貌似这几年国内银行业发生了不下10起事故，但影响面不大，报到也不多，广大群众知道的少。还有一种是保证CP，舍弃A，例如网络故障时只读不写。

另外，虽然上面第二节讲到过我们只能选择CP和AP，无法选择CA。但这句话成立的前提条件是在系统发生了网络故障的情况下。然而，网络故障的概率在系统的整个生命周期中占比是很小的，因此我们在设计的时候，虽然要考虑网络问题下的方案，但也要考虑网络正常情况下的方案，即在网络正常情况下，CA是可以实现的，我们也需要去保证在绝大多数时间下的CA架构。

一个分布式系统无论在CAP三者之间如何权衡，都无法彻底放弃一致性（Consistency），如果真的放弃一致性，那么就说明这个系统中的数据根本不可信，数据也就没有意义，那么这个系统也就没有任何价值可言。所以，无论如何，分布式系统的一致性问题都需要重点关注。

