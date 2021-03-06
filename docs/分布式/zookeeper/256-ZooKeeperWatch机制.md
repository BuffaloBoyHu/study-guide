_**为什么添加Watch**_

> ZooKeeper是用来协调（同步）分布式进程的服务，提供了一个简单高性能的协调内核，用户可以在此之上构建更多复杂的分布式协调功能。

> 多个分布式进程通过ZooKeeper提供的API来操作共享的ZooKeeper内存数据对象ZNode来达成某种一致的行为或结果，这种模式本质上是基于状态共享的并发模型，与Java的多线程并发模型一致，他们的线程或进程都是”共享式内存通信“。Java没有直接提供某种响应式通知接口来监控某个对象状态的变化，只能要么浪费CPU时间毫无响应式的轮询重试，或基于Java提供的某种主动通知（Notif）机制（内置队列）来响应状态变化，但这种机制是需要循环阻塞调用。而ZooKeeper实现这些分布式进程的状态（ZNode的Data、Children）共享时，基于性能的考虑采用了类似的异步非阻塞的主动通知模式即Watch机制，使得分布式进程之间的“共享状态通信”更加实时高效，其实这也是ZooKeeper的主要任务决定的—协调。Consul虽然也实现了Watch机制，但它是阻塞的长轮询。

![](http://dl2.iteye.com/upload/attachment/0121/5326/0983521a-6b2c-3bca-b195-840e844be2ab.png)

* _**ZooKeeper VS JVM**_

  从某种角度来说，可以这样对比（个人看法，可以讨论），ZooKeeper对等于JVM，ZooKeeper包含状态对象（ZNode）和分布式进程的底层执行引擎Zab，而JVM内部包含堆（多线程共享的大量对象存放区域）和多线程执行正确性约束规范JMM（Java内存模型），JMM确保了多线程的执行顺序是正确的。Zab协议使得ZooKeeper的内部修改状态操作直接是有序串行的，而JVM内部则是乱序并行的，需要添加额外的机制才能保证时序（内存屏障、处理器原子指令），而状态读取时，JVM和ZooKeeper都存在直接读取时读到旧数据，但ZooKeeper有Watch机制使得响应式读取更高效，而JVM只能使用底层的内存屏障刷新共享状态，以便其他线程再次读取时获得正确的新数据。

  ZooKeeper提供的接口使得所有的分布式进程的执行都是异步非阻塞的（WaitFree算法），内部是基于Version的CAS操作，而JVM提供了阻塞的和非阻塞的多种接口，有Synchronized、Volatile、AtomicOperations。基于接口之上构建线程或分布式进程之间更复杂的同步或协调功能时，Java并发库直接提供了闭锁、循环栅栏、信号量等同步工具以及基础的抽象队列同步器，而ZooKeeper则需要用户基于接口自行构建各种分布式协调功能（分布式锁、分布式发布订阅、集群成员关系管理）。如下图：

|  | _**ZooKeeper**_ | _**JVM**_ |
| :--- | :--- | :--- |
| _**共享状态对象**_ | ZNode | 堆中对象 |
| _**底层执行模式**_ | Zab顺序执行 | 多处理器并发执行（内存屏障、原子机器指令） |
| _**API接口**_ | Get、Watch\_Get、Cas\_Set、Exist | Synchronized、volatile、final、Atomic |
| _**协调或同步功能**_ | 分布式发布订阅、锁、读写锁 | 并发库同步工具、基于抽象队列同步器构建的同步组件 |

* _**ZooKeeper的Watch架构**_

  Watch的整体流程如下图所示，客户端先向ZooKeeper服务端成功注册想要监听的节点状态，同时客户端本地会存储该监听器相关的信息在WatchManager中，当ZooKeeper服务端监听的数据状态发生变化时，ZooKeeper就会主动通知发送相应事件信息给相关会话客户端，客户端就会在本地响应式的回调相关Watcher的Handler。

![](http://dl2.iteye.com/upload/attachment/0121/5399/90a468b0-870e-3c1a-af6d-43a164b13aa7.png)

* _** ZooKeeper的Watch特性**_

* Watch是一次性的，每次都需要重新注册，并且客户端在会话异常结束时不会收到任何通知，而快速重连接时仍不影响接收通知。

* Watch的回调执行都是顺序执行的，并且客户端在没有收到关注数据的变化事件通知之前是不会看到最新的数据，另外需要注意不要在Watch回调逻辑中阻塞整个客户端的Watch回调
* Watch是轻量级的，WatchEvent是最小的通信单元，结构上只包含通知状态、事件类型和节点路径。ZooKeeper服务端只会通知客户端发生了什么，并不会告诉具体内容。

* _**Watcher接口设计**_

![](http://dl2.iteye.com/upload/attachment/0121/5405/a5939643-5918-36c0-9765-cb008d47ae1b.jpg)

```
如上图所示，Watch被设计成一个接口，任何实现了Watcher接口的类就是一个新的Watcher，Watcher内部包含2个枚举类，一个KeeperState，表示当事件发生时ZooKeeper的状态，另一个是事件发生的类型，主要分为2类（一类是ZNode内容的变化，另一类是ZNode子节点的变化），具体的描述见下表。
```

| _**KeeperState**_ | _**EventType**_ | _**TriggerCondition**_ | _**EnableCalls**_ | _**Desc**_ |
| :--- | :--- | :--- | :--- | :--- |
| SyncConnected\(3\) | None\(-1\) | 客户端与服务器成功建立会话 |  | 此时客户端与服务器处于连接状态 |
| 同上 | NodeCreated\(1\) | Watcher监听的对应数据节点被创建 | Exists | 同上 |
| 同上 | NodeDeleted\(2\) | Watcher监听的对应数据节点被删除 | Exists, GetData, and GetChildren | 同上 |
| 同上 | NodeDataChanged\(3\) | Watcher监听的数据节点的数据内容和数据版本号发生变化 | Exists and GetData | 同上 |
| 同上 | NodeChildrenChanged\(4\) | Watcher监听的数据节点的子节点列表发生变化，子节点内容变化不会触发 | GetChildren | 同上 |
| Disconnected\(0\) | None\(-1\) | 客户端与ZooKeeper服务器断开连接 |  | 此时客户端与服务器处于断开连接的状态 |
| Expried\(-112\) | None\(-1\) | 会话超时 |  | 此时客户端会话失效，通常同时也会收到SessionExpiredException异常 |
| AuthFailed\(4\) | None\(-1\) | 通常有两种情况：1.使用错误的scheme进行权限检查2.SASL权限检查失败 |  | 收到AuthFailedException异常 |

* _**WatchEvent的设计**_

![](http://dl2.iteye.com/upload/attachment/0121/5409/72c8a1e5-e665-3106-b40f-d097e5b18914.png)

如上图所示，WatchEvent有2种表示模式，一种是逻辑表示即WatchedEvent，是直接封装了各种抽象的逻辑状态（KeeperState，EventType），适用于客户端和服务端各自内部处理，另一种是物理表示即封装的更多是底层基础的传输数据结构（int，String），并且实现了序列化接口，主要用来做底层的数据传输。





