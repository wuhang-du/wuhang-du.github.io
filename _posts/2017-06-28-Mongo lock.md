---
layout: post_layout
title: Mongo锁机制与索引的理解
time: 2017年06月28日 星期三
location: 北京
pulished: true
---

对Mongo锁机制以及索引的一些分析与理解

<!--break-->


------

#### **1.锁机制** ####

----

锁机制保证多个客户端读和写时看到的是同样的数据。

2.2 之前，mongo 实例只有一把全局的读写锁。

2.2 之后，实现了粒度更小，数据库级别的锁。同时，对于一些长时间运行的操作，当满足一些条件时，则放弃锁。
  全局锁仍然存在，但是只使用在mongo实例的级别，而且用的很少。我的理解是对全局的admin等操作时，会用到。



我现在使用的版本是mongo 2.6, mongostat会返回一个 locked_db ,这个参数在以后的版本中没有了，之后的版本
是locked ,含义是 the percent of time in a global write lock.

```
mongo 2.2  locked_db

The percent of time in the per-database context-specific lock. mongostat will report
the database that has spent the most time since the last mongostat call with a write
 lock.

This value represents the amount of time that the listed database spent in a locked 
state combined with the time that the mongod spent in the global lock. Because of 
this, and the sampling method, you may see some values greater than 100%

choosing “global” displays a derived metric that adds the percent of time spent in 
the global lock (typically a very small number) plus the percent of time locked by the
 hottest database at the time of measurement, where hottest means “most locked.” 
Because the data is sampled and combined, it is possible to see values over 100%

ps:今天使用createindex({},{'background':true})时，这个值在10s内维持在160%-170%之间，
同时page faults从0增加，看起来是锁住了内存，query等操作从硬盘读数据，内存一直保持
在locked状态，所以升到了100%以上。
```

MongoDB uses a readers-writer lock that allows concurrent reads access to a database 
but gives exclusive access to a single write operation.

The “greediness” of our writes was not only keeping our clients from being able to access
 data (in any of our collections), but causing additional writes to be delayed

正常的Locked_db并没有什么定论，根据实际的场景决定，如果是 写频繁的业务，有可能普遍超过60%，
但是 读频繁的业务，就不会超过10%。

```
wiki上读写锁的伪代码：
Begin Read
Lock r.
Increment b.
If b = 1, lock g.
Unlock r.
End Read
Lock r.
Decrement b.
If b = 0, unlock g.
Unlock r.
Begin Write
Lock g.
End Write
Unlock g.
```


----

####  **2.索引**  ####

----

关于索引，mongo除了单索引之外，还支持复合索引，还有一个神奇的东西叫**索引交集**。

**单索引**： 就是简单的单个索引，这种情况下顺序不重要，mongo会reverse.

**复合索引**：多个field生成一个索引，复合索引有几点需要注意：

> * a,b,c代表字段
> * 1.索引字段的顺序，例如，使用abc生成索引，查询顺序是abc,ab,a的时候都可以用的上，bc,c就不会使用。
> * 2.索引字段的升序(1)降序(-1),如果(a:1,b:-1)生成索引，则(a:1,b:-1),(a:-1,b:1)的查询都可以使用该索引，但是(a:-1,b:-1),(a:1,b:1)就不会使用。

**索引交集**：是mongo提供的一种机制，可以将单个索引联合起来使用。例如2提到的索引升序降序，如果使用
两个单个的(a:1),(b:1)就可以支持4种组合了。


索引交集与复合索引的比较：复合索引必须注重：fileds的顺序，以及fileds 的顺序逆序的组合。然而，索引交集的使用有一个限制，即如果sort部分，需要使用的索引与query不相关的索引，则无法使用索引交集。

```go
{ qty: 1 }
{ status: 1, ord_date: -1 }
{ status: 1 }
{ ord_date: -1 }

{ qty: 1 }
{ status: 1, ord_date: -1 }
{ status: 1 }
{ ord_date: -1 }
no: 不使用索引交集
no: 没用上索引交集, compound index也不支持
db.orders.find({qty:{$gt:10}}).sort( { status: 1 } )
yes: 用上了compound index, 没用索引交集
db.orders.find({qty:{$gt:10}, status: "A" } ).sort( { ord_date: -1 } )
```

  **结论就是需要具体问题具体对待，没有技术银弹**。


```
举个今天的例子：

db.push_log.find({
        'msg.sendTime':{$gt:1496654230780.0,$nin:['']},
        'msg.msgType':{$in:['chat','g_card']},
       $or:[{'msg.recvId':{$in:['wh90013524']}},{'msg.userId':'wh90013524'}]
       }).limit(50).sort({'msg.sendTime':-1}).explain(2)

对于这个例子，两种思路：
第一种：重写查询语句，将or提到顶层，这样的话，就相当于两个不同的查询。

db.push_log.find({
	$or:[{
        'msg.msgType':{$in:['chat','g_card']},
		'msg.recvId':{$in:['wh90013524']}
        'msg.sendTime':{$gt:1496654230780.0,$nin:['']},
	},{
        'msg.msgType':{$in:['chat','g_card']},
		'msg.userId':'wh90013524'
        'msg.sendTime':{$gt:1496654230780.0,$nin:['']},
	}]
}).limit(50).sort({'msg.sendTime':-1}).explain(2)

两个变动：1.or裂开。 2.sendTime 后置。

此时建立 msgType,userId,sendTime  和 msgType,recvId,sendTime,或者建
立单索引，调用索引交集的机制。

第二种办法：不改变现有的语句，修改索引。

经测试:
        删掉所有的自建索引，当第一种情况处理，建立time,type,recvid 和 
time，type,userid, 此时使用的是compound index,并没有触发or机制。
        删掉所有的自建索引，建立type,recvid,time 和 type,userid,time ,
此时使用了caluse 机制，索引可以使用。但是对于or的两部分，两种方案，每种
方案下对两个分支使用了同样的索引1或2。
        删掉所有的自建索引，建立recvid 和 userId ,此时是一种方案下，
两个索引同时使用, or机制正常启用。
```
贴几篇有用的文章：

前两篇是关于锁机制。

[Learn About Lock Percentage: Concurrency in MongoDB](https://www.mongodb.com/blog/post/learn-about-lock-percentage-concurrency-in)

[mongodb-performance-optimization-with-mms](https://www.mongodb.com/blog/post/mongodb-performance-optimization-with-mms)

这几篇是关于索引的讨论。

[复合索引](https://docs.mongodb.com/master/core/index-compound/)

[索引交集](https://docs.mongodb.com/master/core/index-intersection/#previous-versions)

[MongoDB $or + sort + index. How to avoid sorting in memory?](https://stackoverflow.com/questions/43323102/mongodb-or-sort-index-how-to-avoid-sorting-in-memory?answertab=oldest#tab-top)

[Cardinal $ins: MongoDB Query Performance over Ranges](https://blog.mlab.com/2012/06/cardinal-ins/)

[Mongodb: Performance impact of $HINT](https://stackoverflow.com/questions/42871998/mongodb-performance-impact-of-hint/42875033#42875033)
