
[TOC]

《 msql实战优化（一）用leftjoin 代替notin 或notexists》首发[牧马人博客](http://www.luckyhe.com/post/85.html)转发请加此提示

#### 优化背景

`header`表:100w数据量，并且表结构有265个字段。（祖传表设计不想吐槽）

`line`表：行数据明细,数据量90w。(每一条明细有个个headerid关联了`header`表)

`rule`表：规则表，小量数据。

所有关键字段都建了索引。如果没有索引这样的数据量会慢的跟蜗牛一样。

#### 业务需求

我要取得与`line`表中业务规则为rule表的所有`header`数据。

第一个简单需求
```sql

select * from header h join line l on l.headerid=h.headerId
where l.ruleId in(select ruleId from rule) 

```
耗时:2分钟查出了20w数据。
对应索引都用到了。但是数据我觉得还是太慢了。
然后我直接使用count(1) 去查。
最终耗时5s。
最终查出原因是网络原因。数据传输太慢了所以查全量数据导致查了1分钟。

第二个需求接踵而至。

我要查出不是上面的数据的所有头信息。也就是要排除上面所取的。

由于测试网落原因我直接使用count用来统计耗时。

第一想法
```sql

select count(1) from header h where h.headerId not in(
select h.headerId from header h join line l on l.headerid=h.headerId
where l.ruleId in(select ruleId from rule)
)

```
由于测试网落原因我直接使用count用来统计耗时。
耗时:9s。


第二种写法
```sql

select count(1) from header h1 where not exists(
select h.headerId from header h join line l on l.headerid=h.headerId
where l.ruleId in(select ruleId from rule)
and h1.headerid=h.headerId
)

```
耗时:43s。
wtf???
然后在执行了一次40s
后续几乎都是40多秒
这里的差距可能是缓存或者是网络导致的。


第三种写法

```sql

select count(1) from header h1 left join(
select h.headerId from header h join line l on l.headerid=h.headerId
where l.ruleId in(select ruleId from rule)
)h on h1.headerid=h.headerId
where h1.headerid is null

```
耗时:12秒
这个测试数据出的时候有点惊讶，感觉太久了。因为网上一大堆文档都说用`left join`的方式代替以上两种会快点。其实这种并不一定最快，这三种方式的快慢都取觉于数据量。但是`not in`跟`not exists`在大数据的情况下都不推荐。尽量使用表关联的方式。这种算法可以在查询初期通过`where`条件尽量把左表数据压缩掉。并且在关键连接字段加上索引。


#### 分析

我最终使用了三种方法来完成我的业务需求。但是执行速度差距有点大。


1. `not in`
>第一种`not in` 场景使用子查询数据量小的情况。因为我子查询只有20w的数据。但是总表有100w。所以我使用not in的话我使用的是外表的索引所以数据较快。
弊端。子查询里面不能存在null字段。如果有，那么你数据就会准。这种方式的实际情况其实是和字表做`hash`连接。

2. `not exists`
>第二种`not in` 场景使用子查询数据量小的情况。因为我子查询只有20w的数据。但是总表有100w。所以我使用`not exists`的话我使用的是子表的索引。但是我外表数据太大。所以导致速度变慢。
本质:对外表作loop循环，每次loop循环再对内表进行查询。

3. `left join`
>第三种`join` 两表进行关联。数据量为两个表的笛卡尔积。返回左表的全部数据。右边不满足条件的为null。如果左表数据大的话，这样关联数据也不小。所以速度这么慢，属于正常。

Nest Loop Join(嵌套连接循环)
```java
(foreach a as v){
	(foreach b as v1){
		(foreach c as v2){
		}
	}
}
```

#### 总结

通过本次实战，对于`not in`,`not exists`,`left join`的本质更清楚了，写的了一手好语句，表的设计也不能丢，我们主表260个字段，真是操蛋。为此，我把主表的一些关键字段提出来了，在此优化之后，我的sql性能提升了20%，这尼玛也太可观了。以后大家建表一定要注意最好不要超过40个字段。不然等数据量一大，你的sql查询会很操蛋。


