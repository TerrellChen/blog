---
title: RocketMQ-特性篇
date: 2020-05-23 16:45:00
tags:
---
<!-- tag -->

## 1、有序性

在某些特定业务场景，可能对消息的顺序有要求。如数据库binlog同步要求binlog内容一定是有序的，如订单管理中对同一个订单的操作要求也是有序的，否则可能得到预期外的结果。

Topic的队列是我们实现有序性的关键，一个topic中，任意队列内部的所有消息均可以认为是有序的，均满足先进先出的有序条件。

针对有序性的需求，围绕队列展开，通常有两种解决方案。

### 1.1、全局有序

全局有序是指，业务对有序性对要求是一个topic内所有消息全部有序。为了达到全局有序的要求，可以将topic的队列数设置为1，即是说所有消息，均投递至同一个队列中，自然也就达到全局有序的效果。

### 1.2、分组有序

分组有序是指，业务对一个topic内的所有消息按照某种条件水平拆分为不同的分组后，组内有序。已知topic的单个队列天然有序，那么只需要将水平拆分后的一组消息，全部发送至同一个topic的队列，即可实现分组有序。

| 有序级别 | 实现方式                                                     | 适合场景                 |
| -------- | ------------------------------------------------------------ | ------------------------ |
| 全局有序 | topic队列数设置为1，消费者顺序消费                           | binlog同步               |
| 分组有序 | 消息按照分组规则固定投递至topic内的某一个队列，消费者顺序消费 | 订单、商品等按主键id拆分 |

#### 1.2.1、分组有序的投递方式

分组有序的投递方式要求我们，数据根据一定条件投递到指定队列，RocketMQ也提供了指定队列的投递接口，可以分为两类：直接指定MessageQueue，或提供MessageQueue生成算法

```java
public class DefaultMQProducer extends ClientConfig implements MQProducer {
  send(org.apache.rocketmq.common.message.Message, org.apache.rocketmq.common.message.MessageQueue)
  ......
  send(org.apache.rocketmq.common.message.Message, org.apache.rocketmq.client.producer.MessageQueueSelector, java.lang.Object)
  ......
}

public class MessageQueue implements Comparable<MessageQueue>, Serializable {
    private static final long serialVersionUID = 6191200464116433425L;
    private String topic;
    private String brokerName;
    private int queueId;
}

public interface MessageQueueSelector {
    MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
}
```

#### 1.2.2、分组有序的队列获取

在分组有序的投递场景下，业务方会希望MessageQueue是不变的，在MessageQueue不变动的情况下才能确保业务按照条件散列后获得的MessageQueue不变。而Broker的停机升级、故障切换等场景又是不可避免的，实际MessageQueue的变化总在发生。

针对这类情况，RocketMQ在nameserver中提供了一个配置方式，可以将topic的队列数写在nameserver的kv存储中。

```java
org.apache.rocketmq.tools.command.topic.UpdateOrderConfCommand
```

当nameserver存在topic相关的orderConf时，client从nameserver查询到的MessageQueue就不再是nameserver上broker注册的实时信息，而是从orderConf中直接拼装而成，是一个不变的信息。

```java
// org.apache.rocketmq.client.impl.factory.MQClientInstance
public static TopicPublishInfo topicRouteData2TopicPublishInfo(final String topic, final TopicRouteData route) {
  TopicPublishInfo info = new TopicPublishInfo();
  info.setTopicRouteData(route);
  // 从orderConf中拼装
  if (route.getOrderTopicConf() != null && route.getOrderTopicConf().length() > 0) {
    String[] brokers = route.getOrderTopicConf().split(";");
    for (String broker : brokers) {
      String[] item = broker.split(":");
      int nums = Integer.parseInt(item[1]);
      for (int i = 0; i < nums; i++) {
        MessageQueue mq = new MessageQueue(topic, item[0], i);
        info.getMessageQueueList().add(mq);
      }
    }

    info.setOrderTopic(true);
  } else {
    // 从broker注册信息中获取
    ......
    info.setOrderTopic(false);
  }

  return info;
}
```

## 2、消费失败的处理

当消息消费失败时，RocketMQ提供了一些封装机制来降低重试使用成本并提高效率

### 2.1、失败后重试

RocketMQ的消费模式有两种：顺序消费、并发消费。无论是顺序消费、并发消费，都支持单条或多条。

```java
public interface MessageListenerOrderly extends MessageListener {
    ConsumeOrderlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeOrderlyContext context);
}

public interface MessageListenerConcurrently extends MessageListener {
    ConsumeConcurrentlyStatus consumeMessage(final List<MessageExt> msgs,
        final ConsumeConcurrentlyContext context);
}
```

在顺序消费的情况下，消费失败后本轮消费的所有消息将会全部重试

```java
// org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService
public boolean processConsumeResult(
  final List<MessageExt> msgs,
  final ConsumeOrderlyStatus status,
  final ConsumeOrderlyContext context,
  final ConsumeRequest consumeRequest
) {
  ......
    switch (status) {
      case SUCCESS:
        ......
          break;
      case SUSPEND_CURRENT_QUEUE_A_MOMENT:
        ......
          // 检查重试次数
          if (checkReconsumeTimes(msgs)) {
            // consume later
            ......
          } else {
            // commit
            .....
          }
        break;
      default:
        break;
        ......
    }
   
private boolean checkReconsumeTimes(List<MessageExt> msgs) {
  ......
    for (MessageExt msg : msgs) {
      if (msg.getReconsumeTimes() >= getMaxReconsumeTimes()) {
        MessageAccessor.setReconsumeTime(msg, String.valueOf(msg.getReconsumeTimes()));
        // 如果超过重试次数，消息发回broker
        if (!sendMessageBack(msg)) {
          // 发送失败则继续重试（重发）
          suspend = true;
          msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
        }
        // 没有超过重试次数则继续本地重试
      } else {
        suspend = true;
        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
      }
    }
  ......
  return suspend;
}
```

当重试超过最大次数，则发回broker。

在并发消费的情况下，消费失败后分为两种情况

```java
// org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService
public void processConsumeResult(
        final ConsumeConcurrentlyStatus status,
        final ConsumeConcurrentlyContext context,
        final ConsumeRequest consumeRequest
    ) {
   ......
        switch (status) {
            case CONSUME_SUCCESS:
               ......
                break;
            case RECONSUME_LATER:
               ......
                break;
            default:
                break;
        }

        switch (this.defaultMQPushConsumer.getMessageModel()) {
            case BROADCASTING:
            // 广播模式，消费失败忽略
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                    log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
                }
                break;
            case CLUSTERING:
            // 集群模式
                List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                  // 消费失败直接发回broker
                    boolean result = this.sendMessageBack(msg, context);
                    if (!result) {
                        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                        msgBackFailed.add(msg);
                    }
                }

                if (!msgBackFailed.isEmpty()) {
                    consumeRequest.getMsgs().removeAll(msgBackFailed);

                  // 发送失败的部分本地再尝试重试
                    this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
                }
                break;
            default:
                break;
        }

        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
        }
    }
```

广播模式下消费失败的消息将会直接全部忽略，集群模式下失败消息会直接发往broker重试

### 2.2、Broker端处理发回的失败消息

Broker端依旧是SendMessageProcessor处理Consumer发回的消息，但与处理正常投递不同，走到另外一条逻辑

```java
// org.apache.rocketmq.broker.processor.SendMessageProcessor
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                      RemotingCommand request) throws RemotingCommandException {
  ......
    try {
      switch (request.getCode()) {
        case RequestCode.CONSUMER_SEND_MSG_BACK:
          // consumer发回的失败消息
          return this.consumerSendMsgBack(ctx, request);
        default:
          // 正常投递
          ......
      }
    } finally {
      ......
    }
}
```

```java
// org.apache.rocketmq.broker.processor.SendMessageProcessor
private RemotingCommand consumerSendMsgBack(final ChannelHandlerContext ctx, final RemotingCommand request)
  throws RemotingCommandException {
  
  ......
  // 获取订阅信息
  SubscriptionGroupConfig subscriptionGroupConfig =
    this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());
  ......
    // 检查broker全县
  if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
    response.setCode(ResponseCode.NO_PERMISSION);
    response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending message is forbidden");
    return response;
  }
	// 订阅信息中查询并检查重试队列相关的信息
  if (subscriptionGroupConfig.getRetryQueueNums() <= 0) {
    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
  }

  // 生成重试topic名
  String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
  int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums();

  int topicSysFlag = 0;
  if (requestHeader.isUnitMode()) {
    topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
  }

  // 查询（或创建）重试队列topic
  TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
    newTopic,
    subscriptionGroupConfig.getRetryQueueNums(),
    PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);
  ......
    
    
  // 记录msg的原topic
  final String retryTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
  if (null == retryTopic) {
    MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
  }
  msgExt.setWaitStoreMsgOK(false);

  
  int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
	......
  
  // 如果消息已经重试的次数大于最大重试次数（来源于订阅信息），那么直接发到死信队列
  // 顺序消费的情况下，因为在本地重试直到超出最大次数，所以发回broker的消息均满足该条件，直接进入死信队列(DLQ)
  // 并发消费+集群模式下，可能进入死信队列，可能进入重试队列
  if (msgExt.getReconsumeTimes() >= maxReconsumeTimes
      || delayLevel < 0) {
    newTopic = MixAll.getDLQTopic(requestHeader.getGroup());
    queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;

    topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic,
                                                                                                   DLQ_NUMS_PER_GROUP,
                                                                                                   PermName.PERM_WRITE, 0
                                                                                                  );
    if (null == topicConfig) {
      response.setCode(ResponseCode.SYSTEM_ERROR);
      response.setRemark("topic[" + newTopic + "] not exist");
      return response;
    }
  } else {
    // 与延迟相关的属性，将会使消息在一段时间后才能消费，下一节介绍
    if (0 == delayLevel) {
      delayLevel = 3 + msgExt.getReconsumeTimes();
    }

    msgExt.setDelayTimeLevel(delayLevel);
  }
......
  
  // 写入消息
  PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
  ......
  
  return response;
}
```

### 2.3、消费失败重试总结

| 消费类型      | 重试方式   | 发回Broker重新消费                                   |
| ------------- | ---------- | ---------------------------------------------------- |
| 顺序消费      | 本地重试   | 发回Broker时直接进入死信队列，不再消费到             |
| 批量消费-广播 | 忽略       | 无                                                   |
| 批量消费-集群 | 本地不重试 | 发回Broker时根据重试次数，可能进入重试队列并重新消费 |

## 3、延迟消息

某些业务场景不希望生产的消息立即被消费，RocketMQ在一定程度上支持延迟消息。

### 3.1、延迟队列

在RocketMQ在Broker端存在一个SCHEDULE_TOPIC_XXXXtopic，其下有18个队列，分别对应不同的延迟时间，也即是RocketMQ原生只支持预设的18个阈值的延迟消息。

使用方式为

```java
// level范围为"1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"对应的索引
msg.setDelayTimeLevel(int level);
```

### 3.2、Broker端对延迟消息的处理

在存储的时候，

```java
// org.apache.rocketmq.store.CommitLog
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
......
  
    if (msg.getDelayTimeLevel() > 0) {
      if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
        msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
      }

      topic = ScheduleMessageService.SCHEDULE_TOPIC;
      queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

      // Backup real topic, queueId
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
      msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

      msg.setTopic(topic);
      msg.setQueueId(queueId);
    }
  }
......
}
```

如果消息设置了DelayLevel，那么消息真实的topic及queue会被保存到properties中，而消息将会被存储至SCHEDULE_TOPIC_XXXX中对应delayLeve的队列里。

同时Broker需要对队列中的消息进行调度，相关逻辑如下

```java
// org.apache.rocketmq.store.schedule.ScheduleMessageService
class DeliverDelayedMessageTimerTask extends TimerTask {
        private final int delayLevel;
        private final long offset;

        public DeliverDelayedMessageTimerTask(int delayLevel, long offset) {
            this.delayLevel = delayLevel;
            this.offset = offset;
        }

        @Override
        public void run() {
            try {
                if (isStarted()) {
                    this.executeOnTimeup();
                }
            } catch (Exception e) {
               ......
            }
        }
  ......
        public void executeOnTimeup() {
          // 获取delayLevel对应的消费队列
            ConsumeQueue cq =
           ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC,
                    delayLevel2QueueId(delayLevel));

            long failScheduleOffset = offset;

            if (cq != null) {
                SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
                if (bufferCQ != null) {
                    try {
                        long nextOffset = offset;
                        int i = 0;
                        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                      // 遍历消费队列
                        for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                            ......
                              // 计算出消息应该被投递的时间，这里小于0是符合投递条件
                            if (countdown <= 0) {
                                MessageExt msgExt =
                                    ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
                                        offsetPy, sizePy);

                               ......
                            } else {
                              // 不符合投递条件（时间不到），则等待间隔后执行 
                                ScheduleMessageService.this.timer.schedule(
                                    new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
                                    countdown);
                                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                                return;
                            }
                        } // end of for

                       ......
                        return;
                    } finally {

                        bufferCQ.release();
                    }
                } // end of if (bufferCQ != null)
                else {
                  // 没有消息，等待下次
									......
                }
            } // end of if (cq != null)
            ......
        }
  ......
    }
```

对应每个delayLevel存在一个定时任务，存在一个定时任务，读取对应的队列，将到期的消息发送到原队列中以供消费。定时任务的初始化位于DefaultMessageStore中。

### 3.3、支持任意时间延迟的设想

很容易想到时间轮算法，然而RocketMQ并不是时间轮算法一个较好的承载体：或者大量时间刻度导致创建大量的队列、或者频繁的遍历引起较高的IO。

但可以尝试结合时间轮与当前固定队列的方式，仅实现一个分钟内（秒级时间轮），同时为消息增加一个属性，能够将消息的精确时间组合为多次延迟队列+秒级时间轮。

例如现在有1mins、5mins两个延迟队列及一个60s（刻度1s）的时间轮，如果有一条定时17min22s的消息，我们可以令这条消息经历三次5mins的延迟队列，同时经历两次1mins的延迟队列，最后再落入时间轮22s的刻度中。
