# RocketMQ支持事务消息机制



## 事务消费

我们经常支付宝转账余额宝，这是日常生活的一件普通小事，但是我们思考支付宝扣除转账的钱之后，如果系统挂掉怎么办，这时余额宝账户并没有增加相应的金额，数据就会出现不一致状况了。

上述场景在各个类型的系统中都能找到相似影子，比如在电商系统中，当有用户下单后，除了在订单表插入一条记录外，对应商品表的这个商品数量必须减1吧，怎么保证？！在搜索广告系统中，当用户点击某广告后，除了在点击事件表中增加一条记录外，还得去商家账户表中找到这个商家并扣除广告费吧，怎么保证？！等等，相信大家或多或多少都能碰到相似情景。

本质上问题可以抽象为：当一个表数据更新后，怎么保证另一个表的数据也必须要更新成功。

如果是单机系统(数据库实例也在同一个系统上)的话，我们可以用**本地事务**轻松解决：

还是以支付宝转账余额宝为例（比如转账10000块钱），假设有
支付宝账户表：A（id，userId，amount）
余额宝账户表：B（id，userId，amount）
用户的userId=1；

从支付宝转账1万块钱到余额宝的动作分为两步：

1）支付宝表扣除1万：update A set amount=amount-10000 where userId=1;
2）余额宝表增加1万：update B set amount=amount+10000 where userId=1;

如何确保支付宝余额宝收支平衡呢？

有人说这个很简单嘛，可以用事务解决。



```bash
Begin transaction 
  update A set amount=amount-10000 where userId=1;
 update B set amount=amount+10000 where userId=1;
End transaction 
commit;
```

这样确实能解决，如果你使用spring的话一个注解就能搞定上述事务功能。



```cpp
@Transactional(rollbackFor=Exception.class) 

  public void update() { 

      updateATable(); 

//更新A表 

      updateBTable(); 

//更新B表 

  }
```

如果系统规模较小，数据表都在一个数据库实例上，上述本地事务方式可以很好地运行，但是如果系统规模较大，比如支付宝账户表和余额宝账户表显然不会在同一个数据库实例上，他们往往分布在不同的物理节点上，这时本地事务已经失去用武之地。

下面我们来看看比较主流的两种方案：

1.**分布式事务—————— 两阶段提交协议**

两阶段提交协议（Two-phase Commit，2PC）经常被用来实现分布式事务。一般分为协调器TC和若干事务执行者两种角色，这里的事务执行者就是具体的数据库，协调器可以和事务执行器在一台机器上。



![img](https://upload-images.jianshu.io/upload_images/1131487-73d6449a4d7c87a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/765/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/35.png)



我们根据上面的图来看看主要流程：

1） 我们的应用程序（client）发起一个开始请求到TC（transaction）；

2） TC先将**prepare消息**写到本地日志，之后向所有的Si发起prepare消息。以支付宝转账到余额宝为例，TC给A的prepare消息是通知支付宝数据库相应账目扣款1万，TC给B的prepare消息是通知余额宝数据库相应账目增加1w。为什么在执行任务前需要先写本地日志，主要是为了故障后恢复用，本地日志起到现实生活中凭证的效果，如果没有本地日志（凭证），出问题容易死无对证；

3） Si收到prepare消息后，执行具体本机事务，但不会进行commit，如果成功返回yes，不成功返回no。同理，返回前都应把要返回的消息写到日志里，当作凭证。

4） TC收集所有执行器返回的消息，如果所有执行器都返回yes，那么给所有执行器发生送commit消息，执行器收到commit后执行本地事务的commit操作；如果有任一个执行器返回no，那么给所有执行器发送abort消息，执行器收到abort消息后执行事务abort操作。

注：TC或Si把发送或接收到的消息先写到日志里，主要是为了故障后恢复用。如某一Si从故障中恢复后，先检查本机的日志，如果已收到commit，则提交，如果abort则回滚。如果是yes，则再向TC询问一下，确定下一步。如果什么都没有，则很可能在prepare阶段Si就崩溃了，因此需要回滚。

现如今实现基于两阶段提交的分布式事务也没那么困难了，如果使用java，那么可以使用开源软件atomikos([http://www.atomikos.com/)来快速实现。](http://www.atomikos.com/)来快速实现。)

不过但凡使用过的上述两阶段提交的同学都可以发现性能实在是太差，根本不适合高并发的系统。为什么？

1）两阶段提交涉及多次节点间的网络通信，通信时间太长！
2）事务时间相对于变长了，锁定的资源的时间也变长了，造成资源等待时间也增加好多！

正是由于分布式事务存在很严重的性能问题，大部分高并发服务都在避免使用，往往通过其他途径来解决数据一致性问题。

**2.使用消息队列来避免分布式事务**

如果仔细观察生活的话，生活的很多场景已经给了我们提示。

比如在北京很有名的姚记炒肝点了炒肝并付了钱后，他们并不会直接把你点的炒肝给你，而是给你一张小票，然后让你拿着小票到出货区排队去取。为什么他们要将付钱和取货两个动作分开呢？原因很多，其中一个很重要的原因是为了使他们接待能力增强（并发量更高）。

还是回到我们的问题，只要这张小票在，你最终是能拿到炒肝的。同理转账服务也是如此，当支付宝账户扣除1万后，我们只要生成一个凭证（消息）即可，这个凭证（消息）上写着“让余额宝账户增加1万”，只要这个凭证（消息）能可靠保存，我们最终是可以拿着这个凭证（消息）让余额宝账户增加1万的，即我们能依靠这个凭证（消息）完成最终一致性。

那么我们如何可靠保存凭证（消息）有两种方法：

**1.业务与消息耦合的方式**

支付宝在完成扣款的同时，同时记录消息数据，这个消息数据与业务数据保存在同一数据库实例里（消息记录表表名为message）。



```csharp
Begin transaction 
       update A set amount=amount-10000 where userId=1; 
       insert into message(userId, amount,status) values(1, 10000, 1); 
End transaction 
commit;
```

上述事务能保证只要支付宝账户里被扣了钱，消息一定能保存下来。

当上述事务提交成功后，我们通过实时消息服务将此消息通知余额宝，余额宝处理成功后发送回复成功消息，支付宝收到回复后删除该条消息数据。

**2.业务与消息解耦方式**

上述保存消息的方式使得消息数据和业务数据紧耦合在一起，从架构上看不够优雅，而且容易诱发其他问题。为了解耦，可以采用以下方式。

1）支付宝在扣款事务提交之前，向实时消息服务请求发送消息，实时消息服务只记录消息数据，而不真正发送，只有消息发送成功后才会提交事务；

2）当支付宝扣款事务被提交成功后，向实时消息服务确认发送。只有在得到确认发送指令后，实时消息服务才真正发送该消息；

3）当支付宝扣款事务提交失败回滚后，向实时消息服务取消发送。在得到取消发送指令后，该消息将不会被发送；

4）对于那些未确认的消息或者取消的消息，需要有一个消息状态确认系统定时去支付宝系统查询这个消息的状态并进行更新。为什么需要这一步骤，举个例子：假设在第2步支付宝扣款事务被成功提交后，系统挂了，此时消息状态并未被更新为“确认发送”，从而导致消息不能被发送。

优点：消息数据独立存储，降低业务系统与消息系统间的耦合；
缺点：一次消息发送需要两次请求；业务处理服务需要实现消息状态回查接口。

### 那么如何解决消息重复投递的问题？

还有一个很严重的问题就是消息重复投递，以我们支付宝转账到余额宝为例，如果相同的消息被重复投递两次，那么我们余额宝账户将会增加2万而不是1万了(上面讲顺序消费是讲过，这里再提一下)。

为什么相同的消息会被重复投递？比如余额宝处理完消息msg后，发送了处理成功的消息给支付宝，正常情况下支付宝应该要删除消息msg，但如果支付宝这时候悲剧的挂了，重启后一看消息msg还在，就会继续发送消息msg。

`解决方法很简单，在余额宝这边增加消息应用状态表（message_apply），通俗来说就是个账本，用于记录消息的消费情况，每次来一个消息，在真正执行之前，先去消息应用状态表中查询一遍，如果找到说明是重复消息，丢弃即可，如果没找到才执行，同时插入到消息应用状态表（同一事务）` 。



```csharp
for each msg in queue 

Begin transaction 

  select count(*) as cnt from message_apply where msg_id=msg.msg_id; 

  if cnt==0 then 

    update B set amount=amount+10000 where userId=1; 

    insert into message_apply(msg_id) values(msg.msg_id); 

End transaction 

commit;
```

为了方便大家理解，我们再来举一个银行转账的示例（和上一个例子差不多）：

比如，Bob向Smith转账100块。

在单机环境下，执行事务的情况，大概是下面这个样子：



![img](https://upload-images.jianshu.io/upload_images/1131487-15dc9515a4ded0b0.png?imageMogr2/auto-orient/strip|imageView2/2/w/479/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/36.png)



当用户增长到一定程度，Bob和Smith的账户及余额信息已经不在同一台服务器上了，那么上面的流程就变成了这样：



![img](https://upload-images.jianshu.io/upload_images/1131487-631d8fe00900f49e.png?imageMogr2/auto-orient/strip|imageView2/2/w/458/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/37.png)



这时候你会发现，同样是一个转账的业务，在集群环境下，耗时居然成倍的增长，这显然是不能够接受的。那如何来规避这个问题？

**大事务 = 小事务 + 异步**

将大事务拆分成多个小事务异步执行。这样基本上能够将跨机事务的执行效率优化到与单机一致。转账的事务就可以分解成如下两个小事务：



![img](https://upload-images.jianshu.io/upload_images/1131487-6322145e2393f7b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/621/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/38.png)



图中执行本地事务（Bob账户扣款）和发送异步消息应该保证同时成功或者同时失败，也就是扣款成功了，发送消息一定要成功，如果扣款失败了，就不能再发送消息。那问题是：我们是先扣款还是先发送消息呢？

首先看下先发送消息的情况，大致的示意图如下：



![img](https://upload-images.jianshu.io/upload_images/1131487-04a916fe162439ac.png?imageMogr2/auto-orient/strip|imageView2/2/w/618/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/39.png)



存在的问题是：如果消息发送成功，但是扣款失败，消费端就会消费此消息，进而向Smith账户加钱。

先发消息不行，那就先扣款吧，大致的示意图如下：



![img](https://upload-images.jianshu.io/upload_images/1131487-dd355ba80f13cd6a.png?imageMogr2/auto-orient/strip|imageView2/2/w/619/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/40.png)



存在的问题跟上面类似：如果扣款成功，发送消息失败，就会出现Bob扣钱了，但是Smith账户未加钱。

可能大家会有很多的方法来解决这个问题，比如：直接将发消息放到Bob扣款的事务中去，如果发送失败，抛出异常，事务回滚。这样的处理方式也符合“恰好”不需要解决的原则。

RocketMQ支持事务消息，下面来看看**RocketMQ是怎样来实现**的?



![img](https://upload-images.jianshu.io/upload_images/1131487-968b661ff12508f4.png?imageMogr2/auto-orient/strip|imageView2/2/w/621/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/41.png)



RocketMQ第一阶段发送Prepared消息时，会拿到消息的地址，第二阶段执行本地事物，第三阶段通过第一阶段拿到的地址去访问消息，并修改消息的状态。

细心的你可能又发现问题了，如果确认消息发送失败了怎么办？RocketMQ会定期扫描消息集群中的事物消息，如果发现了Prepared消息，它会向消息发送端(生产者)确认，Bob的钱到底是减了还是没减呢？如果减了是回滚还是继续发送确认消息呢？

RocketMQ会根据发送端设置的策略来决定是回滚还是继续发送确认消息。这样就保证了消息发送与本地事务同时成功或同时失败。

那我们来看下RocketMQ源码，是如何处理事务消息的。

客户端发送事务消息的部分(完整代码请查看：rocketmq-example工程下的com.alibaba.rocketmq.example.transaction.TransactionProducer)



```csharp
// =============================发送事务消息的一系列准备工作========================================

// 未决事务，MQ服务器回查客户端

// 也就是上文所说的，当RocketMQ发现`Prepared消息`时，会根据这个Listener实现的策略来决断事务

TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();

// 构造事务消息的生产者

TransactionMQProducer producer = new TransactionMQProducer("groupName");

// 设置事务决断处理类

producer.setTransactionCheckListener(transactionCheckListener);

// 本地事务的处理逻辑，相当于示例中检查Bob账户并扣钱的逻辑

TransactionExecuterImpl tranExecuter = new TransactionExecuterImpl();

producer.start()

// 构造MSG，省略构造参数

Message msg = new Message(......);

// 发送消息

SendResult sendResult = producer.sendMessageInTransaction(msg, tranExecuter, null);

producer.shutdown();
```

接着查看sendMessageInTransaction方法的源码，总共分为3个阶段：发送Prepared消息、执行本地事务、发送确认消息。



```kotlin
//  ================================事务消息的发送过程=============================================

public TransactionSendResult sendMessageInTransaction(.....) {

    // 逻辑代码，非实际代码

    // 1.发送消息

    sendResult = this.send(msg);

    // sendResult.getSendStatus() == SEND_OK

    // 2.如果消息发送成功，处理与消息关联的本地事务单元

    LocalTransactionState localTransactionState = tranExecuter.executeLocalTransactionBranch(msg, arg);

    // 3.结束事务

    this.endTransaction(sendResult, localTransactionState, localException);

}
```

endTransaction方法会将请求发往broker(mq server)去更新事务消息的最终状态：

1.根据sendResult找到Prepared消息 ，sendResult包含事务消息的ID

2.根据localTransaction更新消息的最终状态

如果endTransaction方法执行失败，数据没有发送到broker，导致事务消息的 状态更新失败，broker会有回查线程定时（默认1分钟）扫描每个存储事务状态的表格文件，如果是已经提交或者回滚的消息直接跳过，如果是prepared状态则会向Producer发起CheckTransaction请求，Producer会调用DefaultMQProducerImpl.checkTransactionState()方法来处理broker的定时回调请求，而checkTransactionState会调用我们的事务设置的决断方法来决定是回滚事务还是继续执行，最后调用endTransactionOneway让broker来更新消息的最终状态。

再回到转账的例子，如果Bob的账户的余额已经减少，且消息已经发送成功，Smith端开始消费这条消息，这个时候就会出现消费失败和消费超时两个问题，解决超时问题的思路就是一直重试，直到消费端消费消息成功，整个过程中有可能会出现消息重复的问题，按照前面的思路解决即可。



![img](https://upload-images.jianshu.io/upload_images/1131487-78925b2c101e6422.png?imageMogr2/auto-orient/strip|imageView2/2/w/624/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/42.png)



消费事务消息
这样基本上可以解决消费端超时问题，但是如果消费失败怎么办？阿里提供给我们的解决方法是：人工解决。大家可以考虑一下，按照事务的流程，因为某种原因Smith加款失败，那么需要回滚整个流程。如果消息系统要实现这个回滚流程的话，系统复杂度将大大提升，且很容易出现Bug，估计出现Bug的概率会比消费失败的概率大很多。这也是RocketMQ目前暂时没有解决这个问题的原因，在设计实现消息系统时，我们需要衡量是否值得花这么大的代价来解决这样一个出现概率非常小的问题，这也是大家在解决疑难问题时需要多多思考的地方。

我们需要注意的是，在3.2.6版本中移除了事务消息的实现，所以此版本不支持事务消息。也就是说，消息失败不会进行检查。

下面我们来看一个简单的例子：

Consumer



```csharp
public class Consumer {

    public static void main(String[] args) throws MQClientException {

        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("transaction_producer");

        consumer.setNamesrvAddr("192.168.1.114:9876;192.168.1.115:9876;192.168.1.116:9876;192.168.1.116:9876");

        /**

         * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>

         * 如果非第一次启动，那么按照上次消费的位置继续消费

         */

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTransaction", "*");

        consumer.registerMessageListener(new MessageListenerOrderly() {

            private Random random = new Random();

            @Override

            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {

                //设置自动提交

                context.setAutoCommit(true);

                for (MessageExt msg:msgs){

                    System.out.println(msg+ " , content : "+ new String(msg.getBody()));

                }

                try {

                    //模拟业务处理

                    TimeUnit.SECONDS.sleep(random.nextInt(5));

                }catch (Exception e){

                    e.printStackTrace();

                    return  ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;

                }

                return ConsumeOrderlyStatus.SUCCESS;

            }

        });

        consumer.start();

        System.out.println("consumer start ! ");

    }

}
```

Producer



```java
public class Producer {

    public static void main(String[] args) throws MQClientException, InterruptedException {

        String group_name = "transaction_producer";

        final TransactionMQProducer producer = new TransactionMQProducer(group_name);

        //namesev服务

        producer.setNamesrvAddr("192.168.1.114:9876;192.168.1.115:9876;192.168.1.116:9876;192.168.1.116:9876");

        //事务回查最小并发数

        producer.setCheckThreadPoolMinSize(5);

        //事务回查最大并发数

        producer.setCheckThreadPoolMaxSize(20);

        //队列数

        producer.setCheckRequestHoldMax(2000);

        producer.start();

        //服务器回调producer,检查本地事务分支成功还是失败

        producer.setTransactionCheckListener(new TransactionCheckListener() {

            @Override

            public LocalTransactionState checkLocalTransactionState(MessageExt messageExt) {

                System.out.println("state --" + new String(messageExt.getBody()));

                return LocalTransactionState.COMMIT_MESSAGE;

            }

        });

        TransactionExecuterImpl transactionExecuter = new TransactionExecuterImpl();

        for (int i = 0; i < 2; i++) {

            Message msg = new Message("TopicTransaction",

                    "Transaction" + i,

                    ("Hello RocketMq" + i).getBytes()

            );

            SendResult sendResult = producer.sendMessageInTransaction(msg, transactionExecuter, "tq");

            System.out.println(sendResult);

            TimeUnit.MICROSECONDS.sleep(1000);

        }

        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {

            @Override

            public void run() {

                producer.shutdown();

            }

        }));

        System.exit(0);

    }

}
```

TransactionCheckListenerImpl



```csharp
/**

 * 执行本地事务，由客户端回调

 */

public class TransactionExecuterImpl implements LocalTransactionExecuter{

    @Override

    public LocalTransactionState executeLocalTransactionBranch(Message msg, Object arg) {

        System.out.println("msg=" + new String(msg.getBody()));

        System.out.println("arg = "+arg);

        String tag = msg.getTags();

        if (tag.equals("Transaction1")){

            //这里有一个分阶段提交的概念

            System.out.println("这里是处理业务逻辑，失败情况下进行ROLLBACK");

            return LocalTransactionState.ROLLBACK_MESSAGE;

        }

        return LocalTransactionState.COMMIT_MESSAGE;

        //return LocalTransactionState.UNKNOW;

    }

}
```

我们先启动消费端，然后启动生产端：

在运行之前，我们先来看一下，web控制台的消息：



![img](https://upload-images.jianshu.io/upload_images/1131487-9ff1856e03607652.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

[业余学习之RocketMQ中级篇](http://of2m1l60v.bkt.clouddn.com/2017.12.17/43.png)



运行结果如下：

生产端：

![img](https://upload-images.jianshu.io/upload_images/1131487-c5afa1b9b26fb778.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



消费端：

![img](https://upload-images.jianshu.io/upload_images/1131487-a71ee0d61e62320e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



![img](https://upload-images.jianshu.io/upload_images/1131487-b50def13529a73ed.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



我们发送了两条消息，消费端只收到一条(第一条)，我们在看看控制台：



![img](https://upload-images.jianshu.io/upload_images/1131487-e1847304972e3573.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



我们发现一种生产了四条消息，原因如下：



![img](https://upload-images.jianshu.io/upload_images/1131487-bee5a93c6e55d93b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1089/format/webp)



这就是为什么我们生产了四条消息，最后却只消费了一条。

我们上面的代码还有这么一段：

```csharp
//服务器回调producer,检查本地事务分支成功还是失败

     producer.setTransactionCheckListener(new TransactionCheckListener() {

         @Override

         public LocalTransactionState checkLocalTransactionState(MessageExt messageExt) {

             System.out.println("state --" + new String(messageExt.getBody()));

             return LocalTransactionState.COMMIT_MESSAGE;

         }

     });
```

这一段代码已经不能够实现相应的功能了（阿里把回查接口实现已经给删除了），回查的逻辑已经不进行开源了（3.2.6），商业版的RocketMQ可以实现消息回查（3.0.8版本也有相应的回查代码。有兴趣的可以进行查看源代码）