# 一、背景
Topic 路由配置在 RocketMQ 中是非常重要的信息，源数据存储在 NameServer 端，现阶段client（包括produer、consumer）获取路由信息是通过30秒一次对NameServer的轮训。这样的设计可以保证周期性的获取最新配置信息，但也存在一些bad case：

* 1、如果 client 频繁访问某个不存在的topic，会导致每次请求都会向 nameServer 发送拉取请求，增加网络开销
* 2、某些网管型应用，客户端可能访问很多 topic，而某些 topic 访问一次后，可能不再访问，或者非常低频的访问，但是 client 端在轮训时，每次还是会从 NameServer 拉取路由信息，增加网络开销的同时，僵尸 Topic 也比较占用内存
* 3、路由信息发生变化时，最长要等待30秒才能拿到最新数据

# 二、改造主体思路

保证30秒轮训机制不变，对 Topic 路由数据的通知、存储机制做一定的优化，主要分以下三部分：

* **a、主动通知机制**。<font color='grey'>增加nameServer主动向client推送的机制，作为轮训的补充；但也要注意2个问题</font>
	* 1、数据热点：比如broker重启时，其关联的所有topic都要发送通知，造成数据热点，解决策略是在channel关闭连接或创建连接时，不触发主动通知机制
	* 2、发送主动通知请求时，检查机器的load及内存，如果发现资源紧张，不触发主动通知机制

* **b、Topic过期机制**。<font color='grey'>为client端topic的设置过期机制，如果某topic超过一段时间未被访问，则自动从client端剔除 </font>
* **c、“Topic不存在”**。<font color='grey'>在client端缓存某个Topic“不存在”的状态，保证当client端频繁访问某个不存在Topic时，能够命中本地缓存，减少网络开销</font>

# 三、详细设计
## 3.1、主动通知
NameServer 发现topic路由信息发生了变化，主动通知 Client 拉取最新路由数据

* NameServer 向 Client 发送路由变动的 Topic
* Client 收到消息后向 NameServer 发送拉取该 Topic 最新路由的请求（<font color='grey'>复用已有流程</font>）
* 
![rmq中NameServer轮训Topic路由](https://user-images.githubusercontent.com/19780771/154845583-f704afb9-24c5-4759-b257-6702313fbefb.png)

改造分为 NameServer 端及 Client 端

### 3.1.1、NameServer
所有操作 Topic 路由信息的操作都封装在类`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager`中，通知机制采用异步触发，即每隔1秒轮训所有 Topic，发现有 Topic 路由发生了变化，那么主动通知 Client

#### 3.1.1.1、Topic 与 Channel

从 NameServer 订阅 Topic Route 的客户端很多，当某个 Topic Route 发生变动后，NameServer 如何知道该通知哪些 Client 呢？为此，可新增两个请求：

* `subscribeTopicRouteInfoChanged()`  订阅路由信息变更
* `unsubscribeTopicRouteInfoChanged()`  取消订阅路由信息变更

然后创建一个类似 Topic 路由信息监听器的服务，例如`TopicRouteListener`，在调用`subscribeTopicRouteInfoChanged()`方法查询路由数据时，将 Topic 与 Channel 的映射关系存储下来，这样当某个 Topic 路由信息发生变更时，便可以快速定位到订阅的 Channel，从而向其发送消息变更通知；同理，当取消订阅时，便将该 Channel 从列表中剔除。映射关系可以存储为以下数据结构（<font color='grey'>key 为 Topic，val 为 Channel 集合</font>）：

```
Map<String, Set<ChannelInfo>> topicChannelMap = new ConcurrentHashMap<>();
```


而关于 Producer 端，为了数据一致性考虑，最好保证 “拉取Topic路由” 与 “订阅路由变更” 为同一个 NameServer；而当有多个 NameServer 时，Producer 对于 NameServer 的选择策略为：

1. 随机选择一个 NameServer 并缓存至本地，后续请求均使用该地址
2. 当 NameServer 列表发生变化（包括列表长度及列表内容）时，重新随机选取一个 NameServer 并缓存

因此在 Producer 端，当选择一个新的 NameServer 时，需要向旧 NameServer 发送`unsubscribe`请求，同时向新 NameServer 发送`subscribe`请求；否则如果只发送 `subscribe` 请求，在运行了一段时间后可能出现多个 NameServer 向同一个 Producer 发送 Topic 变更消息，造成资源浪费




#### 3.1.1.2、Topic 路由信息变更及热点规避

当前版本中，涉及 topic route 修改的方法共有 **6** 处：

1. `deleteTopic()` 删除topic信息
2. `registerBroker()` 注册broker
3. `unregisterBroker()` broker主动注销
4. `wipeWritePermOfBrokerByLock()` 修改perm字段
5. `addWritePermOfBrokerByLock()` 同上
6. `onChannelDestroy()` broker 通道被动关闭

在主动通知机制中，比较典型的热点为 Broker 的启动、关闭，因为在某个 Broker 上可能注册了大量的 Topic，一旦启停会触发频繁 IO，故可以主动识别

* **Broker 启动** ： 对应`registerBroker()`方法，但为某个 Broker 创建 Topic 时同样也是调用的此方法，所以可以根据`TopicConfigSerializeWrapper`中 Topic 的数量来判断，如果数量大于 1 的话，则可以识别为 Broker 启动
* **Broker 关闭** ：对应`unregisterBroker() / onChannelDestroy()`方法，这个调用比较直观，在 Broker 关闭时，JVM HOOK发起调用

在其他方法上，可以添加`TopicRouteListener.onChange()`的监听，分别向这些 Channel 发送主动通知；因此，可以简单整理以下表格

| 方法       | 操作           | 
| ------------- |:-------------:| 
| `deleteTopic()`      |    <font color='green'>通知客户端</font>     |   
| `registerBroker()`      |    <font color='grey'>根据请求的 Topic 数量而定</font>      |   
| `unregisterBroker()`      |    <font color='red'>不通知客户端</font>      |   
| `wipeWritePermOfBrokerByLock()`      |    <font color='green'>通知客户端</font>      |   
| `addWritePermOfBrokerByLock()`      |    <font color='green'>通知客户端</font>      |   
| `onChannelDestroy()`      |    <font color='red'>不通知客户端</font>      |   




#### 3.1.1.3、网络通信协议
现版本 Topic 路由信息都是被动响应，即 request-response 模型；本改造继续沿用此设计风格，NameServer 主动通知 Client 哪些 Topics 发生了变动（仅发送 Topic names），消息类型则设定为 Request 模式，增加新的请求码

```
RequestCode.NOTIFY_CLIENT_TOPIC_ROUTE_CHANGED
```
同时新建`RequestHeader`，主要存储 Topic Names，而 body 内容设定为空即可

![简单协议](https://user-images.githubusercontent.com/19780771/154845603-894013a5-2993-4d60-8564-8558892658b2.png)


本机制是对轮训策略的补充，定性为优化改善，发送的结果不必关心，所以发送方式策略设置为 **Oneway**，提高吞吐及性能

#### 3.1.1.4、规避退让
本改造定性为优化，当 NameServer 资源紧张时，希望能够自动关闭功能；可借助JDK提供的`OperatingSystemMXBean`类获取响应的指标：**内存使用率、机器整体负载、网络带宽使用率**


* 当内存使用率、机器负载、网络带宽均低于60%，主动通知的频率为1秒一次
* 当内存使用率、机器负载、网络带宽只要有一项高于60%且低于90%，主动通知的频率降低为10秒一次
* 当内存使用率、机器负载、网络带宽只要有一项高于90%，主动通知临时关闭


### 3.1.2、Client (consumer / producer)
常规的 Client 发送数据，NameServer 响应的流程如下：（request-response模式）
![rmq网络response模式](https://user-images.githubusercontent.com/19780771/154845623-cadc12af-18bc-4005-9e6b-419db8a0eff1.png)


Client 端收到 Netty 的消息后，首先判断类型

* response 类型，即常规模式，交给 ResponseTable 注册的回调函数去处理
* request 类型，即服务端主动调用客户端，类`org.apache.rocketmq.client.impl.ClientRemotingProcessor`负责处理，其根据 Request.code 做了不同的业务分发

为了应对此场景，需要新增一种 Request.code，如下：
![rmq网络request模式](https://user-images.githubusercontent.com/19780771/154845635-d8142fa2-c800-4510-aabc-8d1f616cbe57.png)


## 3.2、Topic路由信息自动过期
### 3.2.1、Producer 端
Producer 端每隔30秒都会通过以下方法拉取所有管理的 Topic Name

```
org.apache.rocketmq.client.impl.producer.MQProducerInner#getPublishTopicList()
```

而 Topic 路由信息在 Producer 端存储的数据结构为

```
ConcurrentMap<String, TopicPublishInfo> topicPublishInfoTable;
```

其中，key 为 Topic Name，val 结构为：

```
public class TopicPublishInfo {
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    private TopicRouteData topicRouteData;
    ......
}
```

#### 改造

所以我们可以在`TopicPublishInfo`类中新加一个字段，比如`lastUpdateTime`，来存储 Topic 上一次访问的时间戳

* **新增**：当首次访问该 Topic 时，新建`TopicPublishInfo`类，将`lastUpdateTime`设置为当前时间
* **修改**：后续有该 Topic 的请求时，更新`lastUpdateTime`字段
	* 该操作可收敛至方法`DefaultMQProducerImpl#tryToFindTopicPublishInfo(topic)` 中
* **删除**：触发时机为 Client 端30秒轮训时
	* 新增方法`MQProducerInner#getTopicListAndRemoveExpired()`，此方法只返回没有过期的 Topic 信息，且将已经过期的 Topic 暂存；判断是否过期的条件是目标 Topic 是否超过5分钟（可配置）还没有访问
	* 从 NameServer 拉取未过期 Topic 的路由信息后，<font color='orange'>向 NameServer 发送已经过期 Topic 的注销消息</font>，目的是当 Topic 路由信息变化后，不需要再通知当前 Client；因此需要新增一种`Request.code`，当 NameServer 收到消息后，解除 Topic 与 Channel 的绑定关系

### 3.2.2、Consumer 端
Consumer的消费分3种形式：

* **Push**：在启动 Consumer 前会指定消费的 Topic `pushConsumer.subscribe(topic, "*")`，即监听指定的 Topic，所以不存在上述提到的问题
* **LitePull**：与 Push 类似，在启动前会指定 Topic；如果 Consumer 想结束直接调用 `consumer.shutdown()`
* **Pull**：在此模式下，虽然会将 Topic 自动注册到`RebalanceImpl#subscriptionInner`中，但30秒轮训时，仅会轮训`DefaultMQPullConsumer#registerTopics`中的 Topic

综上，Consumer 客户端不涉及改造，维持现状

## 3.3、标记不存在的 Topic
### 3.3.1、Producer 端
如果<font color='orange'> Topic 存在，或者 Broker 端允许自动创建 Topic </font>的话，Producer 端会将 Topic 的路由信息缓存至本地；当再次访问此 Topic 时，则直接命中本地的缓存，性能较高，如下图：

![Topic路由信息缓存流程_正常](https://user-images.githubusercontent.com/19780771/154845659-e3e54055-36b3-4428-bdb1-a30b982778ea.png)


但如果是<font color='orange'> Topic 不存在，且 Broker 不允许自动创建 Topic </font>的话，每次请求的流程将会变成如下：

![Topic路由信息缓存流程_异常](https://user-images.githubusercontent.com/19780771/154845671-5eef7621-cd44-4be8-9a81-3c9cc4648354.png)

因此解决思路也比较直观，即将 Topic 不存在的信息也缓存至本地，当频繁访问某个不存在的 Topic 时，可不用每次都发起网络请求

在`org.apache.rocketmq.client.impl.producer.TopicPublishInfo`类中添加 boolean 字段，标记目标 Topic 是否存在，与普通 Topic 一样，同样也会参与30秒轮训及5分钟过期机制

路由数据均存储在`DefaultMQProducerImpl#topicPublishInfoTable`属性中，简单比较一下改造前后，对待不存在 Topic 的处理操作

|        | 不存在的Topic如何处理           | 
| ------------- |:-------------| 
| 改造前     |    新建`TopicPublishInfo`对象，并将其放入`topicPublishInfoTable`中    |   
| 改造后      |   新建`TopicPublishInfo`对象，并将其放入`topicPublishInfoTable`中<br/><font color='orange'>且将`TopicPublishInfo`对象的`topicExist`属性设置为 false</font>      |   

对`TopicPublishInfo`类中的行为判断方法，比如`ok()`不做变动，继续保持改造前的语义，因此整体风险可控

### 3.3.2、Consumer 端
不做改动
