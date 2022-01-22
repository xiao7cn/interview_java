返回上级： 【[Java面试知识体系](../../../../index.md)】→【[中间件](../../../index.md)】→【[Redis](../../index.md)】→【[Redis数据结构](../index.md)】



# Redis中的跳跃表（skiplist）详解



### 前言

跳跃表结构在 **Redis** 中的运用场景只有一个，那就是作为有序列表 **(Zset)** 的使用。跳跃表的性能可以保证在查找，删除，添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这就是跳跃表的长处。跳跃表的缺点就是需要的存储空间比较大，属于利用空间来换取时间的数据结构。接下来我们思考三个问题：

### 思考三个问题

- **跳跃表的底层结构是什么样的，为什么可以支撑它在对数期望时间内完成基本操作（增删改查）？**
- **在跳跃表中，完成一个元素的增删改查的详细过程是怎样的？**
- **利用跳跃表作为底层数据结构的有序列表，在实际的业务场景中有什么运用？**

### 跳跃表结构

跳跃表结构
![跳跃表是长下面这样的(图片来自于维基百科)](https://www.pianshen.com/images/422/f4666fdac87ce12a310c17eda8e94cc6.png)
在跳跃表中，每个跳跃表的节点都会维护着一个 **score** 的值，这个值在跳跃表中是按照大小排好序的。

#### 跳跃表的数据结构源代码

```C
    typedef struct zskiplist {

    // 头节点，尾节点
    struct zskiplistNode *header, *tail;

    // 节点数量
    unsigned long length;

    // 目前表内节点的最大层数
    int level;

} zskiplist;
123456789101112
```

- header 指向了跳跃表的头结点，tail 指向跳跃表的尾节点
- length 表示了跳跃表节点中的数量
- level 表示跳跃表的表内节点的最大层数

#### 跳跃表的节点结构如下图所示

```
typedef struct zskiplistNode {

    // member 对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 这个层跨越的节点数量
        unsigned int span;

    } level[];

} zskiplistNode;
1234567891011121314151617181920212223
```

#### 直观来感受下跳跃表结构

![跳跃表节点图](https://www.pianshen.com/images/223/217a1902d6bcc626ce3fc1383c446e57.png)

- **obj (成员对象)**：对应的是图中的 o1, o2, o3,是用来存储一个节点中的对象的。
- **score (分值)**：对应的是每一个成员对象中的 1.0，2.0 等分数值。
- **后退指针**：这个指针指向的是前面的一个跳表节点。
- **层**：这个结构包括前进指针和记录了跨越的节点数量，这块就是跳跃表的精髓所在。

**跳跃表的基本结构就是上面所展示的部分，接下来我们开始进行分析跳跃表的基础操作过程（增删改查）**

#### 跳跃表增删查改过程

![image](https://www.pianshen.com/images/405/9bb96d6cc29e573fdabb89567efc5415.png)
一个跳跃表的一个节点是 64 层，能够存储的节点数量应该 2^64 个。在源码中是这样的,官方没有其他的解释。

```C
define ZSKIPLIST_MAXLEVEL 64 /* Should be enough for 2^64 elements */
1
```

##### **查找过程**：

按照图中所示，我们现在需要查找的是值为 7 的这个节点。步骤如下：

- 从 head 节点开始，为了演示方便，这里显示的是4层，实际上的是64层。先是降一层到值 4 这个节点的这一层。如果不是所需要的值，那么就再降一层，跳跃到值为 6 的这一层。最后查找到值为 7 。这就是查找的过程，时间复杂度为 O(lg(n))

##### **插入过程**：

插入的过程和查找的过程类似：比如要插入的值为 6

- 从 head 节点开始，先是在 head 开始降层来查找到最后一个比 6 小的节点，等到查到最后一个比 6 小的节点的时候(假设为 5 )。然后需要引入一个**随机层数算法**来为这个节点随机地建立层数。把这个节点插入进去以后，同时更新一遍最高的层数即可。

##### **随机算法**

```c
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
12345678910
```

> Redis 源码中的晋升概率为25%，所以相对来说，Redis 的层高数相对来说是比较扁平化，层高相对较低，所以需要遍历的节点数量会多一些。

##### **删除过程**：

- 删除的过程也是和查找的过程一样，先是找到要删除的那个值，再把这个值给删除，同时把重排一下指针和更新最高的层数。

##### **更新过程**：

- 更新的过程和插入的过程都是是使用着 **zadd** 方法的，先是判断这个 **value** 是否存在，如果存在就是更新的过程，如果不存在就是插入过程。在更新的过程是，如果找到了Value，先删除掉，再新增，这样的弊端是会做两次的搜索，在性能上来讲就比较慢了，在 **Redis 5.0** 版本中，**Redis** 的作者 **Antirez** 优化了这个更新的过程，目前的更新过程是如果判断这个 **value**是否存在，如果存在的话就直接更新，然后再调整整个跳跃表的 score 排序，这样就不需要两次的搜索过程。
  可以看看关于 **Antirez** 这次的[更新优化代码。](https://github.com/antirez/redis/commit/201168368acb3bcabdb01aa191487573409239f5)

### 实际的业务场景

#### Zset 数据结构

![image](https://www.pianshen.com/images/862/57af872369ad2bf37f8885a264394ce6.png)

- 如图所示，Zset 的数据结构是有一个 hash 表和一个跳跃表来结合的，hash 表上存储的是关于 String 的值和 Score 的值，跳跃表是用来辅助 hash 表来实现关于按照 score 来排序的功能。

所以跳跃表的实际运用场景就是 Zset 的实际运用场景

# 

#### Zset的使用示例

```JAVA
//给某个集合增加权重和成员
//成员不可以为重复，权重可以重复，一个集合可以容纳到2^32-1个元素
//增加元素
redis 127.0.0.1:6379> ZADD spacedong 1 redis
redis 127.0.0.1:6379> ZADD spacedong 2 mongodb
redis 127.0.0.1:6379> ZADD spacedong 3 mysql

//获取集合中的元素个数
redis 127.0.0.1:6379> ZCARD spacedong 
"3"

//获取集合中的某个范围的成员
redis 127.0.0.1:6379> ZRANGE spacedong 0 2
1)  "redis"
2)  "mongodb"
3)  "mysql"
12345678910111213141516
```

#### Zset的实际运用场景

- 在 Zset 中使用最多的场景就是涉及到排行榜类似的场景。例如实时统计一个关于分数的排行榜，这个时候可以使用 Redis 中的这个 ZSET 数据结构来维护。
- 涉及到需要按照时间的顺序来排行的业务场景，例如如果需要维护一个问题池，按照时间的先后顺序来维护，这个时候也可以使用 Zset ，把时间当做权重，把问题当做 key 值来进行存取。