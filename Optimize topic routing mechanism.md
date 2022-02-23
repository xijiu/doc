
# Status
* Current State: Proposed
* Authors: [xijiu](https://github.com/xijiu)
* Shepherds: [dongeforever](https://github.com/dongeforever)
* Mailing List discussion: dev@rocketmq.apache.org
* Pull Request: 
* Released: no

# Background & Motivation
## What do we need to do
* Will we add a new module ?
	* no 
* Will we add new APIs ?
	* no 
* Will we add new feature ?
	* 1、增加NameServer端主动通知Client机制 
	* 2、Client缓存Topic不存在的状态  
	* 3、Client 端设置Topic主动过期


## Why should we do that
* Are there any problems of our current project?
	* 1、如果 Client 频繁访问某个不存在的topic，会导致每次请求都会向 nameServer 发送拉取请求，增加网络开销
	* 2、某些网管型应用，客户端可能访问很多 topic，而某些 topic 访问一次后，可能不再访问，或者非常低频的访问，但是 client 端在轮训时，每次还是会从 NameServer 拉取路由信息，增加网络开销的同时，僵尸 Topic 也比较占用内存
	* 3、路由信息发生变化时，最长要等待30秒才能拿到最新数据

* What can we benefit proposed changes?
	* 1、路由信息如果发生了变化，Client 端能够相对实时拿到配置信息
	* 2、定时清空低频Topic，降低 Client 消耗及内存占用

# Goals
* What problem is this proposal designed to solve?
	* 1、Client 频繁访问不存在的 Topic 时，可以命中本地缓存，不需要重复发起网络请求
	*  2、Topic 的路由信息如果发生了变化，Client 端能够相对实时拿到配置信息，而不是等待30秒
	*  3、Client 会定时清空本地使用频率低的 Topic 路由数据
* To what degree should we solve the problem?
	* 上述问题1、3都可以得到解决；
	* 问题2可以得到很大程度的缓解，Topic路由数据变更后，最快在1秒内就可以拿到路由数据

# Non-Goals
* What problem is this proposal NOT designed to solve?
	* 本次设计不会让客户端实时拿到路由数据，而是**准实时**拿到数据，例如1秒 
* Are there any limits of this proposal?
	* 没有

# Changes
## Architecture

### 1、主动通知
NameServer 发现topic路由信息发生了变化，主动通知 Client 拉取最新路由数据

* NameServer 向 Client 发送路由变动的 Topic
* Client 收到消息后向 NameServer 发送拉取该 Topic 最新路由的请求（<font color='grey'>复用已有流程</font>）

![rmq中NameServer轮训Topic路由](https://user-images.githubusercontent.com/19780771/155311627-a21ebdaa-0fce-4d4d-a32a-70d194621a0a.png)


常规的 Client 发送数据，NameServer 响应的流程如下：（request-response模式）

![rmq网络response模式](https://user-images.githubusercontent.com/19780771/155311663-c58664d3-927c-455c-9d2e-95eacd42b513.png)



Client 端收到 Netty 的消息后，首先判断类型

* response 类型，即常规模式，交给 ResponseTable 注册的回调函数去处理
* request 类型，即服务端主动调用客户端，类`org.apache.rocketmq.client.impl.ClientRemotingProcessor`负责处理，其根据 Request.code 做了不同的业务分发

为了应对此场景，需要新增一种 Request.code，如下：

![rmq网络request模式](https://user-images.githubusercontent.com/19780771/155311711-f7e56900-0ae5-4df2-a478-5897f44f010e.png)


### 2、Topic路由信息自动过期

* Producer 端
	* Producer 端每隔30秒都会拉取所有 Topic Names，所以我们可以在`TopicPublishInfo`类中新加一个字段，比如`lastUpdateTime`，来存储 Topic 上一次访问的时间戳；这样每当进行30秒轮训时，便检测哪些 Topic 已经过期，如果 Topic 过期则向 NameServer 发送注销信息

* Consumer 端
	* Consumer 对于 Topic 的处理与 Producer 不同
	* Consumer的消费形式分3种：Push、LitePull、Pull；Push 与 LitePull 在启动前便已经指定好了 Topic，而 Pull 模式仅会轮训 `DefaultMQPullConsumer#registerTopics` 中的 Topic
	* 综上，Consumer 客户端不涉及改造，维持现状


### 3、标记不存在的 Topic

#### Producer 端
如果<font color='orange'> Topic 存在，或者 Broker 端允许自动创建 Topic </font>的话，Producer 端会将 Topic 的路由信息缓存至本地；当再次访问此 Topic 时，则直接命中本地的缓存，性能较高，如下图：
![Topic路由信息缓存流程_正常](https://user-images.githubusercontent.com/19780771/155311747-9a307a2e-f5ed-4eaa-bf13-f0f8452568a8.png)


但如果是<font color='orange'> Topic 不存在，且 Broker 不允许自动创建 Topic </font>的话，每次请求的流程将会变成如下：
![Topic路由信息缓存流程_异常](https://user-images.githubusercontent.com/19780771/155311778-181432d7-423d-45d3-912e-0bba9880eda6.png)


因此解决思路也比较直观，即将 Topic 不存在的信息也缓存至本地，当频繁访问某个不存在的 Topic 时，可不用每次都发起网络请求

#### Consumer 端

不涉及改造，维持现状



## Interface Design/Change
### Method signature/behavior changes
新增两个请求码

```
// Server 端主动发起Topic路由信息变更通知的
public static final int NOTIFY_CLIENT_TOPIC_ROUTE_CHANGED = 328;

// Client 端主动发起注销Topic监听
public static final int UNREGISTER_TOPIC_ROUTE = 329;
```

NameServer 端新增 Topic 路由数据变更的监听服务：当发生注册、注销事件时，修改Topic与Channel的映射关系；当Topic路由数据发生变化时，存储在Map中，等待NameServer轮训线程将变化通知给到Client端

```
/**
 * topic route listener
 */
public interface TopicRouteListener {

    void register(String topic, Channel channel);

    void unregister(String topic, Channel channel);

    void onChange(String topic, TopicRouteData topicRouteData);
}
```


### CLI command changes
无

### Log format or content changes
无

## Compatibility, Deprecation, and Migration Plan
* Are backward and forward compatibility taken into consideration?
	* 是的，考虑了兼容性问题，能完成向后和向前兼容
* Are there deprecated APIs?
	* 无 
* How do we do migration?
	* 正常升级即可 

## Implementation Outline
We will implement the proposed changes by 3 phases. 

## Phase 1 - 主动通知
增加nameServer主动向client推送的机制，作为轮训的补充；但也要注意2个问题

* 1、数据热点：比如broker重启时，其关联的所有topic都要发送通知，造成数据热点，解决策略是在channel关闭连接或创建连接时，不触发主动通知机制
* 2、发送主动通知请求时，检查机器的load及内存，如果发现资源紧张，不触发主动通知机制

### NameServer
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

![简单协议](https://user-images.githubusercontent.com/19780771/155311832-e011b47d-c740-44ea-b733-35b102d0f4c9.png)


本机制是对轮训策略的补充，定性为优化改善，发送的结果不必关心，所以发送方式策略设置为 **Oneway**，提高吞吐及性能

#### 3.1.1.4、规避退让
本改造定性为优化，当 NameServer 资源紧张时，希望能够自动关闭功能；可借助JDK提供的`OperatingSystemMXBean`类获取响应的指标：**内存使用率、机器整体负载、网络带宽使用率**


* 当内存使用率、机器负载、网络带宽均低于60%，主动通知的频率为1秒一次
* 当内存使用率、机器负载、网络带宽只要有一项高于60%且低于90%，主动通知的频率降低为10秒一次
* 当内存使用率、机器负载、网络带宽只要有一项高于90%，主动通知临时关闭

### 3.1.2、Client (consumer / producer)

Client 端收到 Netty 的消息后，首先判断类型

* response 类型，即常规模式，交给 ResponseTable 注册的回调函数去处理
* request 类型，即服务端主动调用客户端，类`org.apache.rocketmq.client.impl.ClientRemotingProcessor`负责处理，其根据 Request.code 做了不同的业务分发

为了应对此场景，如上文所说，需要新增一种 Request.code


## Phase 2 - Topic路由信息自动过期

### Producer 端
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

### Consumer 端
Consumer的消费分3种形式：

* **Push**：在启动 Consumer 前会指定消费的 Topic `pushConsumer.subscribe(topic, "*")`，即监听指定的 Topic，所以不存在上述提到的问题
* **LitePull**：与 Push 类似，在启动前会指定 Topic；如果 Consumer 想结束直接调用 `consumer.shutdown()`
* **Pull**：在此模式下，虽然会将 Topic 自动注册到`RebalanceImpl#subscriptionInner`中，但30秒轮训时，仅会轮训`DefaultMQPullConsumer#registerTopics`中的 Topic

综上，Consumer 客户端不涉及改造，维持现状

## Phase 3 - 标记不存在的 Topic
### 3.3.1、Producer 端

在`org.apache.rocketmq.client.impl.producer.TopicPublishInfo`类中添加 boolean 字段，标记目标 Topic 是否存在，与普通 Topic 一样，同样也会参与30秒轮训及5分钟过期机制

路由数据均存储在`DefaultMQProducerImpl#topicPublishInfoTable`属性中，简单比较一下改造前后，对待不存在 Topic 的处理操作

|        | 不存在的Topic如何处理           | 
| ------------- |:-------------| 
| 改造前     |    新建`TopicPublishInfo`对象，并将其放入`topicPublishInfoTable`中    |   
| 改造后      |   新建`TopicPublishInfo`对象，并将其放入`topicPublishInfoTable`中<br/><font color='orange'>且将`TopicPublishInfo`对象的`topicExist`属性设置为 false</font>      |   

对`TopicPublishInfo`类中的行为判断方法，比如`ok()`不做变动，继续保持改造前的语义，因此整体风险可控

### 3.3.2、Consumer 端
不做改动


# Rejected Alternatives 
无其他备选方案

