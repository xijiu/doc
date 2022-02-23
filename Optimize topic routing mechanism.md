# 1.Background

Topic routing data is very important information in RocketMQ, which meta data is stored on the NameServer. And the client (including producer and consumer) obtains the routing info through the rotation training of the NameServer every 30 seconds. Such design can ensure that the latest configuration information is periodically obtained, but there are also some bad cases:

1. If the client frequently requests a topic that does not exist, it will cause each request to send a pull request to the nameServer, increasing the network overhead
2. For some applications, the client may access many topics, and some topics may no longer be accessed after one visit, or the access may be very infrequent. But the client will still pull routing information from the NameServer every time during round-robin training. While increasing network overhead, zombie topics also occupy more memory
3. When the topic routing information changes, it takes up 30 seconds to get the latest data


# 2.Transformation Ideas
Under the premise that the **30-second rotation is unchanged**, the notification and storage of Topic routing data are optimized to a certain extent, which are divided into the following three parts:

* a. **Active notification**   NameServer actively pushes data to the client; but also pay attention to 2 problems

	*  Hotspot data; For example, when the broker restarts, all related topics send notifications and causing data hotspots. The solution is to not trigger the active notification mechanism when the channel closed or created.
	*  When sending an active notification request, check the load and memory of the machine. If the resource is found to be tight, the active notification will be temporarily closed.

* b. **Topic expired** The client sets an expiration mechanism for the topic. If a topic has not been accessed for a period of time, it will be automatically removed from the client.
* c. **Topic not existed** Cache the "non-existent" status of a topic on the Client to ensure that when the client frequently accesses a topic that does not exist, it can hit the local cache and reduce network overhead.

# 3.Detailed Design

## 3.1 Active Notification
NameServer finds that topic routing has changed, and actively notifies Client to pull the latest routing data

* NameServer send Topic name to Client
* After the client receives the message, it sends a request to the NameServer to pull the latest route data of the topic

![](/Users/likangning/Documents/md/rmq中NameServer轮训Topic路由.png)

### 3.1.1 NameServer
All Topic routing data operations are in class `org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager`, The notification is triggered **async**. All topics are rotated every 1 second. If a topic route is found to change, it will actively notifies the client.

#### 3.1.1.1、Topic and Channel
There are many clients that subscribe Topic Routes from NameServer. When a Topic Route changes, how does NameServer know which Clients to notify? To this end, two new requests can be added:

* `subscribeTopicRouteInfoChanged()`  Subscription routing information changes
* `unsubscribeTopicRouteInfoChanged()`  Unsubscribe from routing information changes

Then create a Topic routing information listener service. When calling the `subscribeTopicRouteInfoChanged()` method, the mapping relationship between Topic and Channel is stored (<font color='grey'>key is Topic, val is Channel collection </font>):

```
Map<String, Set<ChannelInfo>> topicChannelMap = new ConcurrentHashMap<>();
```

As for the Producer side, for data consistency, it is best to ensure that "pulling topic routes" and "subscribing route changes" are the same NameServer; and when there are multiple NameServers, the Producer's selection strategy for NameServers is:

1. Randomly select a NameServer and cache it locally, and subsequent requests will use this address
2. When the NameServer list changes (length or content), a new NameServer is randomly selected and cached

Therefore, when selecting a new NameServer, you need to send an `unsubscribe` request to the old NameServer and a `subscribe` request to the new NameServer; otherwise, if you only send a `subscribe` request, after running for a period of time, multiple NameServers may appear to the same Producer sends Topic change messages

#### 3.1.1.2、Topic routing information change 
In the current version, There are **6** methods that modify the routing data:

1. `deleteTopic()` delete topic route data
2. `registerBroker()`  broker register
3. `unregisterBroker()` broker unregister
4. `wipeWritePermOfBrokerByLock()` modify perm attribute
5. `addWritePermOfBrokerByLock()` modify perm attribute
6. `onChannelDestroy()` broker channel closed

In the active notification, the typical hotspot is the startup and shutdown of the broker, because a large number of topics may be registered on a broker, once the startup and shutdown will trigger frequent IO

* **Broker startup** ： `registerBroker()`, But this method is also called when creating a topic for a broker, so it can be judged according to the number of topics in `TopicConfigSerializeWrapper`, if the number > 1, it's the broker started
* **Broker shutdown** ：`unregisterBroker() / onChannelDestroy()`, when the Broker is closed, the JVM HOOK call


| method       | operation           | 
| ------------- |:-------------:| 
| `deleteTopic()`      |    <font color='green'>notify clients</font>     |   
| `registerBroker()`      |    <font color='grey'>Depends on Topic number</font>      |   
| `unregisterBroker()`      |    <font color='red'>Do not notify clients</font>      |   
| `wipeWritePermOfBrokerByLock()`      |    <font color='green'> notify clients </font>      |   
| `addWritePermOfBrokerByLock()`      |    <font color='green'> notify clients </font>      |   
| `onChannelDestroy()`      |    <font color='red'>Do not notify clients</font>      |   




#### 3.1.1.3、Network protocol
NameServer notify Client which topics have changed (only Topic names), the type is set to Request mode, and a new request code is added

```
RequestCode.NOTIFY_CLIENT_TOPIC_ROUTE_CHANGED
```

![](/Users/likangning/Documents/md/简单协议.png)

This optimization is a supplement to the rotation training strategy. You don't need to care about the results sent, so the sending type can be set to **Oneway**

#### 3.1.1.4 Service degradation
When nameserver resources are tight, we hope the service can be degraded automatically.


### 3.1.2、Client (consumer / producer)
the mode of request-response

![](/Users/likangning/Documents/md/rmq网络response模式_english.png)


When Client receives the message from Netty

* type of response, ResponseTable callback function to handle
* type of request, NameServer actively notify Clients, the class `org.apache.rocketmq.client.impl.ClientRemotingProcessor` is responsible for processing which did different business processing according to Request.code

A new Request.code needs to be added
![](/Users/likangning/Documents/md/rmq网络request模式_english.png)



## 3.2、Topic routing data expires automatically
### 3.2.1、Producer 
The Producer will pull all Topic Names every 30 seconds through the following methods

```
org.apache.rocketmq.client.impl.producer.MQProducerInner#getPublishTopicList()
```

The data structure stored on the Producer is

```
ConcurrentMap<String, TopicPublishInfo> topicPublishInfoTable;
```

key is Topic Name, and val is

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

So we can add a new field to the TopicPublishInfo class, such as `lastUpdateTime`, to store the timestamp of the last access to the Topic

* **Add**: set `lastUpdateTime` to the current timestamp
* **Update**: Update `lastUpdateTime` to the current timestamp
	* `DefaultMQProducerImpl#tryToFindTopicPublishInfo(topic)`
* **Delete**：The trigger timing is the client's 30-second rotation
	* Add method `MQProducerInner#getTopicListAndRemoveExpired()` which  only returns Topic route that has not expired, and stores the Topic that has expired (the condition for judging whether the Topic has expired is whether the Topic has not been accessed for more than 5 minutes)
	* After the topic's routing data is pulled from the NameServer, send the expired Topic to the NameServer. When the NameServer receives the message, unbind the Topic and Channel

### 3.2.2、Consumer
There are 3 types of Consumer:

* **Push**：Subscribe Topic before start, `pushConsumer.subscribe(topic, "*")`
* **LitePull**：Similar to Push, Subscribe Topic before start
* **Pull**：Although topics will be automatically registered in `RebalanceImpl#subscriptionInner`,  but only topics in `defaultmqpullconsumer#registertopics` will be rotated during 30 second rotation

So, the consumer does not change

## 3.3、Mark topic doesn't exist
### 3.3.1、Producer 
If **topic exist, or broker allow create topic automatically**, will hit local cache

![](/Users/likangning/Documents/md/Topic路由信息缓存流程_正常_english.png)

But if topic **NOT** exist, and broker **NOT** allow create topic automatically

![](/Users/likangning/Documents/md/Topic路由信息缓存流程_异常_english.png)

So we need to cache the information that Topic does not exist to the local. Add new boolean attribute to `org.apache.rocketmq.client.impl.producer.TopicPublishInfo`, mark the topic exist or not

### 3.3.2、Consumer
no change


[中文链接](https://github.com/xijiu/doc/blob/main/%E4%BC%98%E5%8C%96rmq%E7%9A%84topic%E8%B7%AF%E7%94%B1%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6.md)
