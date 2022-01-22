# Rabbitmq 的消息确认机制(事务+confirm)



问题：生产者将消息发送出去之后，消息到底有没有到达 rabbitmq 服务器。默认情况下是不知道的。

两种方式：

- AMQP 实现了事务机制
- Confirm 模式

#### 事务机制

- txSelect：用户将当前的 channel 设置成 transaction 模式
- txCommit：用于提交事务
- txRollback：回滚事务

缺点：降低了 rabbitmq 的吞吐量。

**生产者**

```
public class TxSend {
    private static final String QUEUE_NAME = "test_queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        String msg = "hello tx message!";        try {
            channel.txSelect();
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            // 出错测试
            int xx = 1 / 0;
            System.out.println("send: " + msg);
            channel.txCommit();
        } catch (Exception e) {
            channel.txRollback();
            System.out.println("send message rollback.");
        }        channel.close();
        connection.close();
    }
}
```

**消费者**

```
public class TxRecv {
    private static final String QUEUE_NAME = "test_queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        channel.basicConsume(QUEUE_NAME, true, new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("recv[tx] msg: " + new String(body, "utf-8"));
            }
        });
    }
}
```

#### Confirm 模式

**生产者端 confirm 模式的实现原理**

生产者将信道设置成 confirm 模式，一旦信道进入 confirm 模式，所有在该信道上面发布的消息都会被指派一个唯一的 ID(从1开始)，一旦消息被投递到所有的匹配队列之后，broker 就会发送一个确认给生产者（包含消息的唯一ID），这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出，broker 回传给生产者的确认消息中 deliver-tag 域包含了确认消息的序列号，此外 broker 也可以设置 basic.ack 的 multiple 域，表示这个序列号之前的所有消息已经得到了处理。

confirm 模式最大的好处在于它是异步的。

开启 confirm 模式：

```
channel.confirmSelect();
```

编程模式：

- 普通 发一条 waitForConfirms()
- 批量 发一批 waitForConfirms()
- 异步 confirm 模式：提供一个回调方法

**confirm 单条**

```
public class Send1 {
    private static final String QUEUE_NAME = "test_queue_confirm1";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        // 生产者调用 confirmSelect 将 channel 设置成为 confirm 模式 （注意）
        channel.confirmSelect();
        String msg = "hello confirm message!";
        channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
        if (!channel.waitForConfirms()) {
            System.out.println("message send failed.");
        } else {
            System.out.println("message send ok.");
        }
        channel.close();
        connection.close();
    }
}
```

**confirm 批量**

```
public class Send2 {
    private static final String QUEUE_NAME = "test_queue_confirm1";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        // 生产者调用 confirmSelect 将 channel 设置成为 confirm 模式 （注意）
        channel.confirmSelect();        String msg = "hello confirm message batch!";
        // 批量模式
        for (int i = 0; i< 10; i++) {
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
        }
        // 确认
        if (!channel.waitForConfirms()) {
            System.out.println("message send failed.");
        } else {
            System.out.println("message send ok.");
        }
        channel.close();
        connection.close();
    }
}
```

**confirm 异步**

Channel 对象提供的 ConfirmListener() 回调方法只包含 deliveryTag (当前 Channel 发出的消息序号)，我们需要自己为每一个 Channel 维护一个 unconfirm 的消息序号集合，每 publish 一条数据，集合中元素加 1，每回调一次 handleAck 方法，unconfirm 集合删掉相应的一条(multiple=false)或多条(multiple=true)记录。从程序运行效率上看，这个 unconfirm 集合最好采用有序集合 SortedSet 存储结构。

```
public class Send3 {
    private static final String QUEUE_NAME = "test_queue_confirm3";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        // 生产者调用 confirmSelect 将 channel 设置为 confirm 模式
        channel.confirmSelect();        // 未确认的消息标识
        final SortedSet<Long> confirmSet = Collections.synchronizedSortedSet(new TreeSet<Long>());        // 通道添加监听
        channel.addConfirmListener(new ConfirmListener() {
            // 没有问题的 handleAck
            public void handleAck(long l, boolean b) throws IOException {
                if (b) {
                    System.out.println("---handleAck---multiple");
                    confirmSet.headSet(l + 1).clear();
                } else {
                    System.out.println("---handleAck---multiple false");
                    confirmSet.remove(l);
                }
            }
            // handleNack 1s 3s 10s xxx...
            public void handleNack(long l, boolean b) throws IOException {
                if (b) {
                    System.out.println("---handleNack---multiple");
                    confirmSet.headSet(l + 1).clear();
                } else {
                    System.out.println("---handleNack---multiple false");
                    confirmSet.remove(l);
                }
            }
        });        String msg = "sssss";
        while (true) {
            long seqNo = channel.getNextPublishSeqNo();
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            confirmSet.add(seqNo);
        }
    }
}
```

**消费者**

(只需修改 QUEUE_NAME)

```
public class Send3 {
    private static final String QUEUE_NAME = "test_queue_confirm3";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtils.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);        // 生产者调用 confirmSelect 将 channel 设置为 confirm 模式
        channel.confirmSelect();        // 未确认的消息标识
        final SortedSet<Long> confirmSet = Collections.synchronizedSortedSet(new TreeSet<Long>());        // 通道添加监听
        channel.addConfirmListener(new ConfirmListener() {
            // 没有问题的 handleAck
            public void handleAck(long l, boolean b) throws IOException {
                if (b) {
                    System.out.println("---handleAck---multiple");
                    confirmSet.headSet(l + 1).clear();
                } else {
                    System.out.println("---handleAck---multiple false");
                    confirmSet.remove(l);
                }
            }
            // handleNack 1s 3s 10s xxx...
            public void handleNack(long l, boolean b) throws IOException {
                if (b) {
                    System.out.println("---handleNack---multiple");
                    confirmSet.headSet(l + 1).clear();
                } else {
                    System.out.println("---handleNack---multiple false");
                    confirmSet.remove(l);
                }
            }
        });        String msg = "sssss";
        while (true) {
            long seqNo = channel.getNextPublishSeqNo();
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            confirmSet.add(seqNo);
        }
    }
}
```

