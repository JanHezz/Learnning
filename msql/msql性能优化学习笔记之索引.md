

《 MySQL性能优化－－数据类型的选择》首发[橙寂博客](http://www.luckyhe.com/post/67.html)转发请加此提示

# 《高性能MySQL》读书笔记－－索引



索引（在MySQL中也叫做键<key>），是存储引擎用于快速找到记录的一种数据结构。

写在前面：

索引对查询的速度有着至关重要的影响，理解索引也是进行数据库性能调优的起点。考虑如下情况，假设数据库中一个表有10^6条记录，DBMS的页面大小为4K，并存储100条记录。如果没有索引，查询将对整个表进行扫描，最坏的情况下，如果所有数据页都不在内存，需要读取10^4个页面，如果这10^4个页面在磁盘上随机分布，需要进行10^4次I/O，假设磁盘每次I/O时间为10ms(忽略数据传输时间)，则总共需要100s(但实际上要好很多很多)。如果对之建立B-Tree索引，则只需要进行log100(10^6)=3次页面读取，最坏情况下耗时30ms。这就是索引带来的效果，很多时候，当你的应用程序进行SQL查询速度很慢时，应该想想是否可以建索引。

索引优化应该是对查询性能优化最有效的手段了，索引能够轻易将查询性能提高几个数量级，”最优“的索引有时比一个”好的“索引性能要好两个数量级。创建一个真正”最优“的索引经常要重写查询。

进入正题：

## **1.索引基础**

在MySQL中，存储引擎用一本书的“索引”找到对应页码类似的方法使用索引，其先在索引中找到对应值，然后根据匹配的索引记录找到对应的数据行。

索引可以包含一个或多个列的值。如果索引包含多个列，那么列的顺序也十分重要，因为MySQL只能高效地使用索引的最左前缀列。

## **2.索引与优化**

### 1、选择索引的数据类型

MySQL支持很多数据类型，选择合适的数据类型存储数据对性能有很大的影响。通常来说，可以遵循以下一些指导原则：

(1)越小的数据类型通常更好：越小的数据类型通常在磁盘、内存和CPU缓存中都需要更少的空间，处理起来更快。</br>
(2)简单的数据类型更好：整型数据比起字符，处理开销更小，因为字符串的比较更复杂。在MySQL中，应该用内置的日期和时间数据类型，而不是用字符串来存储时间；以及用整型数据类型存储IP地址。</br>
(3)尽量避免NULL：应该指定列为NOT NULL，除非你想存储NULL。在MySQL中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值。


### 2、索引入门

对于任何DBMS，索引都是进行优化的最主要的因素。对于少量的数据，没有合适的索引影响不是很大，但是，当随着数据量的增加，性能会急剧下降。
如果对多列进行索引(组合索引)，列的顺序非常重要，MySQL仅能对索引最左边的前缀进行有效的查找。例如：
假设存在组合索引it1c1c2(c1,c2)，查询语句select * from t1 where c1=1 and c2=2能够使用该索引。查询语句select * from t1 where c1=1也能够使用该索引。但是，查询语句select * from t1 where c2=2不能够使用该索引，因为没有组合索引的引导列，即，要想使用c2列进行查找，必需出现c1等于某值。

#### 2.1、索引的类型

**索引是在存储引擎中实现的，而不是在服务器层中实现的。**所以，每种存储引擎的索引都不一定完全相同，并不是所有的存储引擎都支持所有的索引类型。

##### 2.1.1、B-Tree索引

B－Tree：每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。B－Tree通常意味着所有的值都是按顺序存储的，并且每一个叶子页到根的距离相同，很适合查找范围数据。

![](http://og0sybnix.bkt.clouddn.com/sp170330_233553.png)
假设有如下一个表：

```mysql
CREATE TABLE People (
   last_name varchar(50)    not null,
   first_name varchar(50)    not null,
   dob        date           not null,
   gender     enum('m', 'f') not null,
   key(last_name, first_name, dob)
);
```

 其索引包含表中每一行的`last_name`、`first_name`和`dob`列。其结构大致如下：

![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-01.JPG) 

 索引存储的值按索引列中的顺序排列。**可以利用B-Tree索引进行全关键字、关键字范围和关键字前缀查询**，当然，如果想使用索引，你必须保证按索引的最左边前缀(leftmost prefix of the index)来进行查询。
 
(1)匹配全值(Match the full value)：对索引中的所有列都指定具体的值。例如，上图中索引可以帮助你查找出生于1960-01-01的Cuba Allen。</br>
(2)匹配最左前缀(Match a leftmost prefix)：你可以利用索引查找last name为Allen的人，仅仅使用索引中的第1列。</br>
(3)匹配列前缀(Match a column prefix)：例如，你可以利用索引查找last name以J开始的人，这仅仅使用索引中的第1列。</br>
(4)匹配值的范围查询(Match a range of values)：可以利用索引查找last name在Allen和Barrymore之间的人，仅仅使用索引中第1列。</br>
(5)匹配部分精确而其它部分进行范围匹配(Match one part exactly and match a range on another part)：可以利用索引查找last name为Allen，而first name以字母K开始的人。</br>
(6)仅对索引进行查询(Index-only queries)：如果查询的列都位于索引中，则不需要读取元组的值。(覆盖索引)
由于B-树中的节点都是顺序存储的，所以可以利用索引进行查找(找某些值)，也可以对查询结果进行ORDER BY。

当然，使用B-tree索引有以下一些限制：

**(1) 查询必须从索引的最左边的列开始，否则无法使用索引。关于这点已经提了很多遍了。例如你不能利用索引查找在某一天出生的人。</br>
(2) 不能跳过某一索引列。例如，你不能利用索引查找last name为Smith且出生于某一天的人。</br>
(3) 存储引擎不能使用索引中范围条件右边的列。例如，如果你的查询语句为WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23'，则该查询只会使用索引中的前两列，因为LIKE是范围查询。**

##### 2.1.2、Hash索引

**哈希索引基于哈希表实现，只有精确索引所有列的查询才有效。对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码，哈希码是一个较小的值，并且不同键值的行计算出来的哈希码也不一样。哈希索引将所有的哈希存储在索引中，同时在哈希表中保存指向每个数据的指针。**

**MySQL中，只有Memory存储引擎显示支持hash索引，是Memory表的默认索引类型**，尽管Memory表也可以使用B-Tree索引。Memory存储引擎支持非唯一hash索引，这在数据库领域是罕见的，如果多个值有相同的hash code，索引把它们的行指针用链表保存到同一个hash表项中。
假设创建如下一个表：

```mysql
CREATE TABLE testhash (

   fname VARCHAR(50) NOT NULL,

   lname VARCHAR(50) NOT NULL,

   KEY USING HASH(fname)

) ENGINE=MEMORY;

```


包含的数据如下：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-02.JPG)

假设索引使用`hash`函数`f( )`，如下：

```mysql
f('Arjen') = 2323
f('Baron') = 7437
f('Peter') = 8784
f('Vadim') = 2458
```

此时，索引的结构大概如下：

![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-03.JPG) 

**哈希索引中存储的是：哈希值+数据行指针 **

Slots是有序的，但是记录不是有序的。当你执行
`mysql> SELECT lname FROM testhash WHERE fname='Peter';`
MySQL会计算’Peter’的hash值，然后通过它来查询索引的行指针。因为`f('Peter') = 8784`，MySQL会在索引中查找8784，得到指向记录3的指针。
因为索引自己仅仅存储很短的值，所以，索引非常紧凑。Hash值不取决于列的数据类型，一个TINYINT列的索引与一个长字符串列的索引一样大。
Hash索引有以下一些限制：
**(1)由于索引仅包含hash code和记录指针，所以，MySQL不能通过使用索引避免读取记录。但是访问内存中的记录是非常迅速的，不会对性造成太大的影响。(2)不能使用hash索引排序。(3)Hash索引不支持键的部分匹配，因为是通过整个索引值来计算hash值的。(4)Hash索引只支持等值比较**，例如使用=，IN( )和<=>。对于WHERE price>100并不能加速查询。

**(5)访问Hash索引的速度非常快，除非有很多哈希冲突（不同的索引列值却有相同的哈希值）。当出现哈希冲突的时候，存储引擎必须遍历链表中所有的行指针，逐行进行比较，直到找到所有符合条件的行。(6)如果哈希冲突很多的话，一些索引维护操作的代价也会很高。当从表中删除一行时，存储引擎要遍历对应哈希值的链表中的每一行，找到并删除对应行的引用，冲突越多，代价越大。**



InnoDB引擎有一个特殊的功能叫做“自适应哈希索引”。当InnoDB注意到某些索引值被使用得非常频繁时，它会在内存中基于B-Tree索引 上再创建一个哈希索引，这样就上B-Tree索引也具有哈希索引的一些优点，比如快速的哈希查找。

**创建哈希索引**：如果存储引擎不支持哈希索引，则可以模拟像InnoDB一样创建哈希索引，这可以享受一些哈希索引的便利，例如只需要很小的索引就可以为超长的键创建索引。

思路很简单：在B-Tree基础上创建一个伪哈希索引。这和真正的哈希索引不是一回事，因为还是使用B-Tree进行查找，但是它使用哈希值而不是键本身进行索引查找。你需要做的就是在查询的where子句中手动指定使用哈希函数。这样实现的缺陷是需要维护哈希值。可以手动维护，也可以使用触发器实现。

如果采用这种方式，记住不要使用SHA1和MD5作为哈希函数。因为这两个函数计算出来的哈希值是非常长的字符串，会浪费大量空间，比较时也会更慢。SHA1和MD5是强加密函数，设计目标是最大限度消除冲突，但这里并不需要这样高的要求。简单哈希函数的冲突在一个可以接受的范围，同时又能够提供更好的性能。

如果数据表非常大，CRC32会出现大量的哈希冲突，CRC32返回的是32位的整数，当索引有93000条记录时出现冲突的概率是1%。

**处理哈希冲突**：当使用哈希索引进行查询时，必须在where子句中包含常量值。

2.1.3、空间(R-Tree)索引
MyISAM支持空间索引，主要用于地理空间数据类型，例如GEOMETRY。
2.1.4、全文(Full-text)索引
全文索引是MyISAM的一个特殊索引类型，它查找的是文本中的关键词主要用于全文检索。

**索引的优点：**

最常见的B-Tree索引，按照顺序存储数据，所以MYSQL可以用来做order by和group by操作。因为数据是有序的，所以B-Tree也就会将相关的列值存储在一起。最后，因为索引中存储了实际的列值，所以某些查询只使用索引就能够完成全部查询。总结下来索引有如下三个优点：

**1，索引大大减小了服务器需要扫描的数据量**

**2，索引可以帮助服务器避免排序和临时表**

**3，索引可以将随机IO变成顺序IO**

索引三星系统：

**一星：索引将相关的记录放到一起**

**二星：索引中的数据顺序和查找中的排列顺序一致**

**三星：索引中的列包含了查询中需要的全部列**

**索引是最好的解决方案吗？**

索引并不总是最好的工具。总的来说只有索引帮助存储引擎快速查找到记录的好处大于其带来的额外工作时，索引才是有效的。

对于非常小的表，大部分情况下简单的全表扫描更高效；

对于中到大型的表，索引就非常有效。

但对于特大型的表，建立和使用索引的代价将随之增长。这种情况下需要一种技术可以直接区分出查询需要的一组数据，而不是一条记录一条记录地匹配。例如使用分区技术。

如果表的数量特别多，可以建立一个元数据信息表，用来查询需要用到的某些特性。例如执行那些需要聚合多个应用分布在多个表的数据的查询，则需要记录“哪个用户的信息存储在哪个表中”的元数据，这样在查询时就可以直接忽略那些不包含指定用户信息的表。



### 3、高性能的索引策略

#### 3.1不能使用索引的案例

以下两个查询无法使用索引：

1）表达式：  select actor_id from sakila.actor where actor_id+1=5;

2）函数参数：select ... where TO_DAYS(CURRENT_DATE) - TO_DAYS(date_col)<=10;

独立的列是指索引列不能是表达式的一部分，也不是是函数的参数。例如以下两个查询无法使用索引：

1）表达式：  select actor_id from sakila.actor where actor_id+1=5;

2）函数参数：select ... where TO_DAYS(CURRENT_DATE) - TO_DAYS(date_col)<=10;

#### 3.2前缀索引和索引选择性

通常可以索引开始的部分字符，这样可以大大节约索引空间，从而提高索引效率。但这样也会降低索引的选择性。索引的选择性是指，不重复的索引值（基数）和数据表中的记录总数（#T）的比值，范围从1/#T之间。索引的选择性越高则查询效率越高，因为选择性高的索引可以让MYSQL在查找时过滤掉更多的行。

唯一索引的选择性是
1，这是最好的索引选择性，性能也是最好的。

一般情况下某个前缀的选择性也是足够高的，足以满足查询性能。对于BLOB、TEXT或者很长的VARCHAR类型的列，必须使用前缀索引，因为MYSQL不允许索引这些列的完整长度。

决窍在于要选择足够长的前缀以保证较高的选择性，同时又不能太长（以便节约空间）。前缀应该足够长，以使得前缀索引的选择性接近于索引整个列。换句话说，前缀的“基数”应该接近于完整列的“基数”。

为了决定前缀的合适长度，需要找到最常见的值的列表，然后和最常见的前缀列表进行比较。例如以下查询：

select count(*) as cnt,city from sakila.city_demo group by city order by cnt desc limit 10;

select count(*) as cnt,left(city,7) as perf  from sakila.city_demo group by city order by cnt desc limit 10;

直到这个前缀的选择性接近完整列的选择性。

计算合适的前缀长度的另一个方法就是计算完整列的选择性，并使前缀的选择性接近于完整列的选择性，如下：

select  count(distinct city)/count(*) from sakila.city_demo;

select  count(distinct left(city,7))/count(*) from sakila.city_demo;

前缀索引是一种能使索引更小、更快的有效办法，但另一方面也有其缺点：MYSQL无法使用前缀索引做order by和group by，也无法使用前缀索引做覆盖扫描。

#### 3.3多列索引

一个多列索引与多个列索引MYSQL在解析执行上是不一样的，如果在explain中看到有索引合并，应该好好检查一下查询的表和结构是不是已经最优。

#### 3.4选择合适的索引列顺序

**对于如何选择索引的顺序有一个经验法则：将选择性最高的列放在索引最前列。**

当不需要考虑排序和分组时，将选择性最高的列放在前面通常是最好的。然后，性能不只是依赖于所有索引列的选择性（整体基数），也和查询条件的具体值有关，也就是和值的分布有关。这和前面介绍的选择前缀的长度需要考虑的地方一样。可能需要根据那些运行频率最高的查询来调整索引列的顺序，让这种情况下索引的选择性最高。

使用经验法则要注意不要假设平均情况下的性能也能代表特殊情况下的性能，特殊情况可能会摧毁整个应用的性能（当使用前缀索引时，在某些条件值的基数比正常值高的时候）。

#### 3.5、聚簇索引(Clustered Indexes)

这里讲的'聚簇索引'与'非聚族索引'不是一种具体的索引类型，而是一种数据存储的方式。

**聚簇索引保证关键字的值相近的元组存储的物理位置也相同（所以字符串类型不宜建立聚簇索引，特别是随机字符串，会使得系统进行大量的移动操作），且一个表只能有一个聚簇索引。**因为由存储引擎实现索引，所以，并不是所有的引擎都支持聚簇索引。目前，只有solidDB和InnoDB支持。
聚簇索引的结构大致如下：

**叶子页包含了行的全部数据，但是节点页只包含了索引列。**

**二级索引叶子节点保存的不是指行的物理位置的指针，而是行的主键值。这意味着通过二级索引查找行，存储引擎需要找到二级索引的叶子节点获取对应的主键值，然后根据这个值去聚簇索引中查找到对应的行。这里做了重复的工作：两次B－TREE查找而不是一次。**![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-04.JPG)

 注：叶子页面包含完整的元组，而内节点页面仅包含索引的列(索引的列为整型)。一些DBMS允许用户指定聚簇索引，但是MySQL的存储引擎到目前为止都不支持。**InnoDB对主键建立聚簇索引。如果你不指定主键，InnoDB会用一个具有唯一且非空值的索引来代替。如果不存在这样的索引，InnoDB会定义一个隐藏的主键，然后对其建立聚簇索引。**一般来说，DBMS都会以聚簇索引的形式来存储实际的数据，它是其它二级索引的基础。

##### 3.5.1、InnoDB和MyISAM的数据布局的比较

为了更加理解聚簇索引和非聚簇索引，或者primary索引和second索引(MyISAM不支持聚簇索引)，来比较一下InnoDB和MyISAM的数据布局，对于如下表：

```mysql
CREATE TABLE layout_test (
   col1 int NOT NULL,
   col2 int NOT NULL,
   PRIMARY KEY(col1),
   KEY(col2)
);
```

 假设主键的值位于1---10,000之间，且按随机顺序插入，然后用OPTIMIZE TABLE进行优化。col2随机赋予1---100之间的值，所以会存在许多重复的值。

###### (1)    MyISAM的数据布局

其布局十分简单，**MyISAM按照插入的顺序在磁盘上存储数据**，如下：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-05.JPG)

 注：左边为行号(row number)，从0开始。因为元组的大小固定，所以MyISAM可以很容易的从表的开始位置找到某一字节的位置。
MyISAM建立的primary key的索引结构大致如下：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-06.JPG)

 注：MyISAM不支持聚簇索引，**索引中每一个叶子节点仅仅包含行号(row number)，且叶子节点按照col1的顺序存储**。
来看看col2的索引结构：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-07.JPG)

 实际上，在MyISAM中，primary key和其它索引没有什么区别。Primary key仅仅只是一个叫做PRIMARY的唯一，非空的索引而已，**叶子节点按照col2的顺序存储**。

###### (2)    InnoDB的数据布局

InnoDB按聚簇索引的形式存储数据，所以它的数据布局有着很大的不同。它存储表的结构大致如下：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-08.JPG)

 注：**聚簇索引中的每个叶子节点包含primary key的值，事务ID和回滚指针(rollback pointer)——用于事务和MVCC，和余下的列**(如col2)。
**相对于MyISAM，InnoDB的二级索引与聚簇索引有很大的不同。InnoDB的二级索引的叶子包含primary key的值，而不是行指针(row pointers)，这样的策略减小了移动数据或者数据页面分裂时维护二级索引的开销，因为InnoDB不需要更新索引的行指针。**其结构大致如下：
![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-09.JPG)

 聚簇索引和非聚簇索引表的对比：

 ![img](http://images.cnblogs.com/cnblogs_com/hustcat/mysql/mysql02-10.JPG)

#####  3.5.2、按primary key的顺序插入行(InnoDB)

如果你用InnoDB，而且不需要特殊的聚簇索引，一个好的做法就是使用**代理主键**(surrogate key)——独立于你的应用中的数据。**最简单的做法就是使用一个AUTO_INCREMENT的列，这会保证记录按照顺序插入，而且能提高使用primary key进行连接的查询的性能。**应该尽量避免随机的聚簇主键，例如，字符串主键就是一个不好的选择，它使得插入操作变得随机。

 

####  3.6、覆盖索引(Covering Indexes)

覆盖索引是一种非常强大的工具，能大大提高查询性能。设计优秀的索引应该考虑到整个查询，而不单单的where条件部分。索引确实是一种查找数据的高效方式，但是MYSQL也可以使用索引来直接获取列的数据，这样就不再需要读取数据行。索引的叶子节点中已经包含要查询的数据，那么就没有必要再回表查询了，**如果索引包含满足查询的所有数据，就称为覆盖索引。**

只需要读取索引而不用读取数据有以下一些优点：</br>
**(1)索引项通常比记录要小，所以MySQL访问更少的数据；</br>(2)索引都按值的大小顺序存储，相对于随机访问记录，需要更少的I/O；</br>(3)大多数据引擎能更好的缓存索引。比如MyISAM只缓存索引。</br>(4)覆盖索引对于InnoDB表尤其有用，因为InnoDB使用聚集索引组织数据，如果二级索引中包含查询所需的数据，就不再需要在聚集索引中查找了。覆盖索引不能是任何索引，只有B-TREE索引存储相应的值。而且不同的存储引擎实现覆盖索引的方式都不同，并不是所有存储引擎都支持覆盖索引(Memory和Falcon就不支持)。**</br>
对于索引覆盖查询(index-covered query)，使用EXPLAIN时，可以在Extra一列中看到“Using index”。例如，在sakila的inventory表中，有一个组合索引(store_id,film_id)，对于只需要访问这两列的查询，MySQL就可以使用索引，如下：

**在大多数引擎中，只有当查询语句所访问的列是索引的一部分时，索引才会覆盖。但是，InnoDB不限于此，InnoDB的二级索引在叶子节点中存储了primary key的值。因此，sakila.actor表使用InnoDB，而且对于是last_name上有索引，所以，索引能覆盖那些访问actor_id的查询**，如：

> **（同时查询actor_id[主键]与last_name[索引字段]）**
>

 

```mysql
mysql> EXPLAIN SELECT actor_id, last_name
    -> FROM sakila.actor WHERE last_name = 'HOPPER'\G
*************************** 1. row ***************************
           id: 1
 select_type: SIMPLE
        table: actor
         type: ref
possible_keys: idx_actor_last_name
          key: idx_actor_last_name
      key_len: 137
          ref: const
         rows: 2
        Extra: Using where; Using index
```

 

#### 3.7、利用索引进行排序

**MySQL中，有两种方式生成有序结果集：一是使用filesort，</br>二是按索引顺序扫描。如果explain出来的type列的值为“index”，则说明MYSQL使用了索引扫描来做排序。利用索引进行排序操作是非常快的，因为只需要从一条索引记录移动到紧接着的下一条记录。但如果索引不能覆盖查询所需的全部列，那就不得不每扫描一条索引记录就回表查询一次对应的行，这基本上都是随机IO，因此按索引顺序读取的速度通常要比顺序地全表扫描慢，尤其是在IO密集型的工作负载时。**
</br>
**而且可以利用同一索引同时进行查找和排序操作。当索引的顺序与ORDER BY中的列顺序相同且所有的列是同一方向(全部升序或者全部降序)时，可以使用索引来排序。如果查询是连接多个表，仅当ORDER BY中的所有列都是第一个表的列时才会使用索引。其它情况都会使用filesort文件排序。**

 

```mysql
create table actor(
actor_id int unsigned NOT NULL AUTO_INCREMENT,
name      varchar(16) NOT NULL DEFAULT '',
password        varchar(16) NOT NULL DEFAULT '',
PRIMARY KEY(actor_id),
 KEY     (name)
) ENGINE=InnoDB
insert into actor(name,password) values('cat01','1234567');
insert into actor(name,password) values('cat02','1234567');
insert into actor(name,password) values('ddddd','1234567');
insert into actor(name,password) values('aaaaa','1234567');

mysql> explain select actor_id from actor order by actor_id \G
*************************** 1. row ***************************
           id: 1
 select_type: SIMPLE
        table: actor
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 4
        Extra: Using index
1 row in set (0.00 sec)
 
mysql> explain select actor_id from actor order by password \G
*************************** 1. row ***************************
           id: 1
 select_type: SIMPLE
        table: actor
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 4
        Extra: Using filesort
1 row in set (0.00 sec)
 
mysql> explain select actor_id from actor order by name \G
*************************** 1. row ***************************
           id: 1
 select_type: SIMPLE
        table: actor
         type: index
possible_keys: NULL
          key: name
      key_len: 18
          ref: NULL
         rows: 4
        Extra: Using index
1 row in set (0.00 sec)
```

 **当MySQL不能使用索引进行排序时，就会利用自己的排序算法(快速排序算法)在内存(sort buffer)中对数据进行排序，如果内存装载不下，它会将磁盘上的数据进行分块，再对各个数据块进行排序，然后将各个块合并成有序的结果集（实际上就是外排序，使用临时表）**。

对于filesort，MySQL有两种排序算法。
(1)两次扫描算法(Two passes)
**实现方式是先将需要排序的字段和可以直接定位到相关行数据的指针信息取出，然后在设定的内存（通过参数sort_buffer_size设定）中进行排序，完成排序之后再次通过行指针信息取出所需的Columns。**
注：该算法是4.1之前采用的算法，它需要两次访问数据，尤其是**第二次读取操作会导致大量的随机I/O操作。另一方面，内存开销较小。**
(2)一次扫描算法(single pass)
**该算法一次性将所需的Columns全部取出，在内存中排序后直接将结果输出。**
注：从 MySQL 4.1 版本开始使用该算法。它**减少了I/O的次数，效率较高，但是内存开销也较大。如果我们将并不需要的Columns也取出来，就会极大地浪费排序过程所需要的内存。在 MySQL 4.1 之后的版本中，可以通过设置 max_length_for_sort_data 参数来控制 MySQL 选择第一种排序算法还是第二种。当取出的所有大字段总大小大于 max_length_for_sort_data 的设置时，MySQL 就会选择使用第一种排序算法，反之，则会选择第二种。为了尽可能地提高排序性能，我们自然更希望使用第二种排序算法，所以在 Query 中仅仅取出需要的 Columns 是非常有必要的。当对连接操作进行排序时，如果ORDER BY仅仅引用第一个表的列，MySQL对该表进行filesort操作，然后进行连接处理，此时，EXPLAIN输出“Using filesort”；否则，MySQL必须将查询的结果集生成一个临时表，在连接完成之后进行filesort操作，此时，EXPLAIN输出“Using temporary;Using filesort”。**

 ![img](data:image/png;base64,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OuBUxG22DpA0lh99jGI85Vovf1mPb8HplrKsVry1FNjryWqnLlJpR57yuNp1O+7nZ/v6omtMa0eYtiWn4dLnbGPRbX+Zzqbja0cT5a/3cUvVcK9jrJo62m0kZD3bgL1Mn5XChC46LwAlV4T9TRqzsv3Gqne8RfLjhJd7nOjnE/S98SQWjZmEH6bd9vHrJvJNWtTm4WgwGi6mXjySAAmQwGgkIL7zQpmM2Iw09yd03nN4+/wpuROM8Jgtd87C7CvWADgVt713ox4C6n+lBktu/zWjBBIgARIgARIggZFK4Otf3aYwz1vQMM9IrbTR+7lLtsfpvzNXwOQrGnCo+QnvBvf5zBYlh7n6LIkZSYAESIAESIAESEARGPHDpv1pSNu4mnzF62g/Pdc8m76VYs81yhq+6ptI5iIBEiABEiABEtjKCUibopBh03eapm7lqFh9EiABEiABEiABEhh6Al/7ylY8bDr0+KkBCZAACZAACZAACRRHoOBh0+22tQcZiyuEqUmABEiABEiABEiABAaGwLbbhAp723RgiqMUEiABEiABEiABEiCB/hLodamQ/hbA/CRAAiRAAiRAAiRAAgNHwH4pcuCkUhIJkAAJkAAJkAAJkMCgEKDxNihYKZQESIAESIAESIAEBocAjbfB4UqpJEACJEACJEACJDAoBGi8DQpWCiUBEiABEiABEiCBwSFA421wuFIqCZAACZAACZAACQwKARpvg4KVQkmABEiABEiABEhgcAjQeBscrpRKAiRAAiRAAiRAAoNCgMbboGClUBIgARIgARIgARIYHAI03gaHK6WSAAmQAAmQAAmQwKAQoPE2KFgplARIgARIgARIgAQGhwCNt8HhSqkkQAIkQAIkQAIkMCgEaLwNClYKJQESIAESIAESIIHBIUDjbXC4UioJkAAJkAAJkAAJDAqBITbe0mitDCEUqkRr2tQvifqQCAshVJ/UgSadDnfi79D57bQii5FhyzXy+3M0cuthNOuPtELyJutVnR0UhWTKSjOIeqdbUSnao7IVThNmlc8AEiABEiABEiCBgSIwxMZbdjWS9RHERXA4hlRbVXYCT8guaFgYVSHxDsegSrc2KxnRhWgo9WQYURfp1kpE4mHEUhn0imKoalbagK5UDOHuRtS5FvhQacNySYAESIAESGDUExhexluyHhFpuUWR6GpAtt2lDJlM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)

#### 3.8、索引与加锁

索引对于InnoDB非常重要，因为它可以让查询锁更少的元组。这点十分重要，因为MySQL 5.0中，InnoDB直到事务提交时才会解锁。有两个方面的原因：首先，即使InnoDB行级锁的开销非常高效，内存开销也较小，但不管怎么样，还是存在开销。其次，对不需要的元组的加锁，会增加锁的开销，降低并发性。
**InnoDB仅对需要访问的元组加锁，而索引能够减少InnoDB访问的元组数。但是，只有在存储引擎层过滤掉那些不需要的数据才能达到这种目的。一旦索引不允许InnoDB那样做（即达不到过滤的目的），MySQL服务器只能对InnoDB返回的数据进行WHERE操作，此时，已经无法避免对那些元组加锁了：InnoDB已经锁住那些元组，服务器无法解锁了。**
来看个例子：

```mysql
create table actor(
actor_id int unsigned NOT NULL AUTO_INCREMENT,
name      varchar(16) NOT NULL DEFAULT '',
password        varchar(16) NOT NULL DEFAULT '',
PRIMARY KEY(actor_id),
 KEY     (name)
) ENGINE=InnoDB
insert into actor(name,password) values('cat01','1234567');
insert into actor(name,password) values('cat02','1234567');
insert into actor(name,password) values('ddddd','1234567');
insert into actor(name,password) values('aaaaa','1234567');

SET AUTOCOMMIT=0;
BEGIN;
SELECT actor_id FROM actor WHERE actor_id < 4
AND actor_id <> 1 FOR UPDATE;
```

 该查询仅仅返回2---3的数据，实际已经对1---3的数据加上排它锁了。**InnoDB锁住元组1是因为MySQL的查询计划仅使用索引进行范围查询（而没有进行过滤操作，WHERE中第二个条件已经无法使用索引了）**：

```mysql
mysql> EXPLAIN SELECT actor_id FROM test.actor
    -> WHERE actor_id < 4 AND actor_id <> 1 FOR UPDATE \G
*************************** 1. row ***************************
           id: 1
 select_type: SIMPLE
        table: actor
         type: index
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 4
        Extra: Using where; Using index
1 row in set (0.00 sec)
 
mysql>
```

 表明存储引擎从索引的起始处开始，获取所有的行，直到actor_id<4为假，服务器无法告诉InnoDB去掉元组1。
为了证明row 1已经被锁住，我们另外建一个连接，执行如下操作：

 

```mysql
SET AUTOCOMMIT=0;
BEGIN;
SELECT actor_id FROM actor WHERE actor_id = 1 FOR UPDATE;
```

 该查询会被挂起，直到第一个连接的事务提交释放锁时，才会执行（这种行为对于基于语句的复制(statement-based replication)是必要的）。
如上所示，**当使用索引时，InnoDB会锁住它不需要的元组。更糟糕的是，如果查询不能使用索引，MySQL会进行全表扫描，并锁住每一个元组，不管是否真正需要。**