# godis中Stream实现

首先，我们先来看看redis stream 使用：推荐阅读文章"[挑战Kafka！Redis5.0重量级特性Stream尝鲜](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=2653549949&idx=1&sn=7f6c4cf8642478574718ed0f8cf61409&chksm=813a64e5b64dedf357cef4e2894e33a75e3ae51575a4e3211c1da23008ef79173962e9a83c73&mpshare=1&scene=1&srcid=0717FcpVc16q9rNa0yfF78FU#rd "超链接title")"<br>
我们使用哈希链表来实现队列，它将队列划分为多个哈希桶，并使用链表将桶中的元素连接在一起。这个数据结构在Redis里面就是使用的Hashmap + Linkedlist结构实现的。

在Go语言中实现这个数据结构的话，我们可以使用一个哈希表和一个双端链表来实现。哈希表中的每个键对应的值都是一个双端链表，每个节点代表一个Redis Stream的元素。

具体来说，每个哈希桶都是一个双端链表，元素通过哈希函数映射到对应的哈希桶中，然后插入到该桶对应的链表中。这样就可以在O(1)时间内插入新元素到Redis Stream中，同时支持快速的查找和删除操作。

同时，哈希链表的队列还支持范围查询，这是Redis Stream的一个重要特性。通过使用一个双向迭代器（类似于游标）来记录当前读取和写入位置，就可以很容易地实现范围查询功能。例如，我们可以从一个时间戳开始，向后或向前遍历链表，直到达到另一个时间戳或者达到列表的末尾。

这个数据结构的优点包括：

- 高性能：插入、删除、查找、范围查询等操作都可以在O(1)时间内完成，而且在大多数情况下都非常高效。
- 可扩展性：由于哈希链表可以很容易地扩展到大规模的工作负载，因此它非常适合处理具有大量数据的应用程序。
- 可定制性：由于哈希链表是一个非常通用的数据结构，因此可以通过调整和定制它来适应各种不同的应用场景。

缺点包括:

- 内存消耗：由于哈希链表需要同时保存元数据和索引信息，因此它们可能需要占用大量的内存。这可能成为处理大量数据时的瓶颈。
- 性能下降：在处理大量数据时，由于哈希链表需要进行哈希计算和链表操作，因此性能可能会出现下降。

让我们来看下具体的结构体
```go
package database
type Node struct {
    Key   string  // 键
    Value string  // 值
    Prev  *Node   // 前一个节点
    Next  *Node   // 后一个节点
}

// 定义哈希链表结构体
type HashList struct {
    Buckets    map[int]*Node // 哈希桶
    Size       int           // 哈希桶的数量
    Head       *Node         // 头节点
    Tail       *Node         // 尾节点
    Current    *Node         // 当前节点
}

// 定义哈希链表队列结构体
type HashMapQueue struct {
    Buckets      []*HashList   // 哈希链表队列（由多个哈希链表组成）
    Size         int           // 哈希链表队列的大小
    RangeMin     string        // 范围查询的最小值
    RangeMax     string        // 范围查询的最大值
    CurrentIndex int           // 当前哈希链表队列的索引
    CurrentNode  *Node         // 当前节点
}
```