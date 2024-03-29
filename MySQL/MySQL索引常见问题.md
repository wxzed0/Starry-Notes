# MySQL索引常见问题

[TOC]

## 使用与否

MySQL每次只使用一个索引，与其说 数据库查询只能用一个索引，倒不如说，和全表扫描比起来，去分析两个索引 B+树更耗费时间，所以where A=a and B=b 这种查询使用（A，B）的组合索引最佳，B+树根据（A，B）来排序。

- 主键，unique字段

- 和其他表做连接的字段需要加索引

- 在where 里使用 >, >=, = , <, <=, is null 和 between等字段。

- 使用不以通配符开始的like，where A like ‘China%’

- 聚合函数里面的 MIN()， MAX()的字段

- order by 和 group by字段

  

## 何时不使用索引

- 表记录太少
- 数据重复且分布平均的字段（只有很少数据的列）；
- 经常插入、删除、修改的表要减少索引
- text，image 等类型不应该建立索引，这些列的数据量大（加入text的前10个字符唯一，也可以对text前10个字符建立索引）
- MySQL能估计出全表扫描比使用索引更快的时候，不使用索引



## 索引失效

- 组合索引为使用最左前缀，例如组合索引（A，B），where B = b 不会使用索引

- like未使用最左前缀，where A like "%China"，使用like查询时以%开头会导致索引失效索引失效，后缀有%时，索引有效

- 搜索一个索引而在另一个索引上做 order by， where A = a order by B，只会使用A上的索引，因为**查询只使用一个索引**。

- **or会使索引失效**。如果**只有当or左右查询字段均为索引时，才会生效。**例如 where A = a1 or A = a2（生效），where A=a or B = b （失效）

- 在索引列上的操作，**函数**upper()等，or、！ = （<>）,not in 等索引失效，例如`select * from table_name where a != 1`

- 在索引上进行计算导致失效 例如`select * from table_name where a + 1 = 2`

- 在索引的类型上进行数据类型的**隐形转换**，会导致索引失效，例如字符串一定要加引号，假设 `select * from table_name where a = '1'`会使用到索引，如果写成`select * from table_name where a = 1`则会导致索引失效。varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。

- 索引字段上使用 is null/is not null判断时会导致索引失效，例如`select * from table_name where a is null`



## 索引使用场景

- 对于中大型表建立索引非常有效，对于非常小的表，一般全部表扫描速度更快些。
- 对于超大型的表，建立和维护索引的代价也会变高，这时可以考虑分区技术。
- 如何表的增删改非常多，而查询需求非常少的话，那就没有必要建立索引了，因为维护索引也是需要代价的。
- 一般不会出现再where条件中的字段就没有必要建立索引了。
- 多个字段经常被查询的话可以考虑联合索引。
- 字段多且字段值没有重复的时候考虑唯一索引。
- 字段多且有重复的时候考虑普通索引。



## 索引优化 & 慢查询优化

### 建索引的几大原则

1. 最左前缀匹配原则，非常重要的原则，**mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配**，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。

2. **=和in可以乱序**，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。
3. **尽量选择区分度高的列作为索引**，区分度的公式是count(distinct col)/count(*)，表示**字段不重复的比例**，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录。
4. **索引列不能参与计算，保持列“干净”**，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)。
5. **尽量的扩展索引，不要新建索引**。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。尽量使用短索引。
6. 最适合索引的列是**where后面出现的列或者连接句子中指定的列**，而不是出现在select关键字后面的选择列表的列



### 优化细节

- 当使用索引列进行查询的时候，尽量不要使用表达式，把计算放到业务层而不是数据库层
- 尽量使用主键查询，而不是其它索引，因为主键查询不会触发回表操作
- 使用索引扫描来进行排序
- union、all、in、or都能使用索引，但是推荐使用in

- 范围列可以使用到索引

- 强制类型转换会让索引失效，进行全表查询