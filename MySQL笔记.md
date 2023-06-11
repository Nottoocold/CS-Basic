## 1. 用户管理及权限管理

命令行登陆参数

```sql
mysql -h [hostname|hostip] -P [port] -u [username] -p [password] [schema] -e [sql-statement]
```

### 1.1 创建用户

核心数据库**mysql**,中的user表记录了所有的用户数据。

创建用户：

```mysql
create user 'username'@'hostname' identified by 'password';
```

修改用户:

```mysql
update mysql.use set use = '...' where user and host; flush privileges;
```

删除用户:

```mysql
drop user user@host; or delete from user where user = '...' and host = '..';
```

修改密码:

- 设置当前用户自己的密码

```mysql
- alter mysql.user user() identified by 'password';
- set password='....';
```

- 设置别人的密码

```mysql
- alter mysql.user 'user'@'host' identified by 'password';
- set password for 'user'@'host' = 'password';
```

手动设置密码过期：

```mysql
alter user 'user'@'host' password expire;
```

用户仍然可以登陆数据库，但是不能进行其他操作，除非重新设置了新密码。
MySQL使用全局变量 default_password_lifetime 默认是0，表示禁用密码过期；可以是正整数，表示N天后密码过期。

### 1.2 权限管理

查看用户权限

```mysql
show grants [for 'user'@'host'];
```

#### 1.2.1 直接赋予权限

```mysql
grant 权限1,权限2... on 数据库名.表名 to 'user'@'host' [identified by 'pwd'];
```

#### 1.2.2 收回权限

```mysql
revoke 权限1,权限2... on 数据库名.表名 from 'user'@'host';
```

#### 1.2.3 如何管理的

MySQL是用一张权限表来控制权限的。**mysql数据库，user表，db表，table_priv,column_privi,proc_privi**。

#### 1.2.4 角色的创建和使用

角色是对权限的“封装”，将角色赋予给具体的用户。

```mysql
- create role 'rolename'@'host';
- grant privileges on schema to role;
```

对角色的crud与角色的语法大致相同。

将角色的权限赋予给用户

```mysql
grant role to user;
```

当角色赋予后，需要**激活**

- ```mysql
  set default [role_name] all to user1,user2...;
  ```

- `系统变量activate_all_roles_on_login默认为OFF,set global .. = ON`

#### 1.2.5 启动命令与选项组

![](http://img.zqqiliyc.love/mysql/202212061845077.png)

## 2. MySQL的逻辑架构

- 连接管理-连接层
- 解析优化-服务层
- 存储引擎-引擎层

MySQL是一个C/S架构。

## 3. 存储引擎

插件式存储引擎。就是表的类型，以前叫做表处理器，后改名为存储引擎，功能就是接收上层传下来的命令，然后对表中数据进行提取和写入操作。

### 3.1 InnoDB

1. 具备外键支持功能的事务存储引擎。被设计用来处理大量的**短期事务**，可以确保事务的完整提交和回滚。

2. 除了增加和查询外，还需要更新和删除则优先选择InnoDB。

3. 除非有其他也别需要，否则应该优先选择InnoDB。

4. 可以处理巨大数据量，并发量。

5. 支持行级锁。

6. 表结构文件.frm，数据文件.ibd

7. 处理的效率差一些，对内存要求高。

### 3.2 MyISAM

主要的非事务存储引擎。

1. 不支持事务，外键，行级锁。

2. 查询，增加效率高。

3. 节省资源，消耗少，简单业务

4. 全文索引，压缩，空间函数

## 4. 索引

索引是在存储引擎中实现的。

进行数据查询时，首先查看数据查询条件是否命中某条索引，符合则使用索引进行扫描，否则使用全表扫描。

### 4.1 索引的数据结构

- 索引是帮助mysql高效查找数据的数据结构。**有效减少IO次数**。

- 索引的本质：**索引就是一个满足特定排序算法的数据结构**，这些数据结构以某种方式指向数据，这样就可以在这种数据结构的基础上，进行高效的查找。

- 索引的创建和维护需要开销，并且随着数据量的增大，开销越发明显。

- 查询快，但是更新操作是需要些时间的。

指对主键构成的索引称为聚集索引，其他的为非聚集索引。

- 聚集索引的数据存放在叶结点上。索引即数据，数据即索引。InnoDB会自动创建，存储引擎自动构建一颗B+树。
  
  - 数据访问更快，因为聚集索引将数据和索引存放在一起。
  - 聚集索引对主键和范围查找非常快。
  - 节省了大量IO，因为是数据是有序的，每次载入内存的是一页，很大概率有目标数据。
  - 按主键递增顺序性能很高。

非聚集索引的结构大体与聚集索引相同，不同点是叶结点不存储真实的数据，只存储索引列和主键值，当查询的不仅有索引列时，需要进行回表查询，即重新按照查询到的主键值查询聚集索引。

联合索引，即将多个列一起当作索引项构建数据结构，按索引列声明的顺序进行构建，如c1,c2,c3,则当c1相同时，按c2排列，当c2相同时，按c3排列，以此类推；叶结点也不含真实数据。

### 4.2 B+树的注意事项

- 根页面的位置万年不动。
- 内节点中目录项记录的唯一性。(列+主键)
- 一个页面最少两条记录。。。

## 5 InnoDB数据存储结构

### 5.1 数据库的存储结构：页

磁盘与内存交换数据的基本单位。mysql中一个页大小16KB。这些页物理上可以不连续，逻辑上是双向链表。每个数据页中的记录维护一张页目录表，当查询时，对页目录表（页目录是对数据分组后的数组，每个元素(slot)取每组中的最大值）进行二分查找。

页的上层结构是区(Extent)，InnoDB中一个区可以存放64个页，即1MB。

再往上是段(Segment)，由一个区或多个区组成，区在文件系统上是一个连续分配的空间,不过在段中区与区之间可以是不连续的。**段是数据库的分配单位，不同类型的数据库对象，以不同的段形式存在**，比如创建表或索引时，会产生一个表段或索引段。

表空间（tablespace）是一个逻辑容器（.ibd文件），存储的内容是段对象，在一个表空间中可以存放一个段或多个段，但是一个段只能属于一个表空间。数据库由一个或多个表空间组成，表空间从管理上可分为：系统表空间、用户表空间、撤销表空间和临时表空间。

### 5.2 页的内部结构

数据页16KB大小的存储空间可分为7个部分。如下图所示。

![](http://img.zqqiliyc.love/mysql/202304260042492.png)

- File_Header又可被细分如下属性：

![](http://img.zqqiliyc.love/mysql/202304260042474.png)

- File_Trailer 文件尾有8字节，前4字节代表页的校验和；后4字节代表页面最后被修改的日志序列位置(LSN)。

- 页目录。对数据分组后进行记录，记录中每一项都是分组数据的最大值。有序。对页目录进行二分粗略地查询检索出目的分组，然后在分组中对真实数据组成的单链表遍历查找。

行格式：compact。

变长字段列表（倒序 2B），NULL值列表（bit），记录头信息（5字节），真实数据隐藏列(row_id 6B),(tx_id 6B),(roll_ptr 7B),(真实数据)。

区的作用是让数据页之间大体上连续存储，可以顺序IO，提高性能。

段的作用，当进行范围查询时，若不区分叶结点和非页节点的话，统统将节点放入一个区中，那么范围查询的性能就会有所下降，因为可能需要加载多次其他页，而若区分的话，将页子结点放入一个区，非页节点放入一个区，将页节点组成的区的集合称为叶子节点段，非页节点称为非页节点段。常见的段有：数据段，索引段和回滚段。

## 6. 索引

### 6.1 索引分类

- 从功能逻辑上说，分为普通，唯一，主键，全文索引。

- 从物理实现上，聚簇索引和非聚簇索引。

- 按照作用字段上分，单列索引和联合索引。

### 6.2 创建索引

- **[fulltext]|[unique]** **index** index_name (**column_name(len)**)

```sql
create table `blog`(
    id int unsigned auto_increment primary key,
    title varchar(100) not null,
    content text not null,
    fulltext index fidx_title_con(title,content) 
);

# 普通模糊查询
# select * from blog where title like '%查询字符串%';
# 创建全文索引后的模糊查询
select * from blog where match(title,content) against('查询字符')
```

全文索引的是大量数据，最好先添加数据，再创建索引。

```mysql
- alter table add **[fulltext|unique] index** idx_name (col_name(len)[asc|desc]);
- create index **[fulltext|unique]** idx_name on **table_name**(col_name(len)[asc|desc]); 
```

### 6.3 删除索引

```mysql
- alter table table_name drop index idx_name;
- drop index idx_name on table_name;
```

### 6.4 索引的设计原则

#### 6.4.1 创建索引的场景

- 字段的值有唯一性限制
- 频繁作为where查询条件的列
- 经常group by 或 order by 的列
- update,delete的where条件列
- distinct 字段需要创建索引
- 多表join连接时，创建索引注意事项
  - 连接表的数量尽量不要超过3个
  - 对where条件创建索引
  - 对用于连接的字段创建索引
- 使用列的类型小的创建索引
- 使用字符串的前缀创建索引
- 区分度高的列适合创建索引
- 使用最频繁的列放到联合索引的左侧
- 在多个字段需要同时创建索引的情况下，优先考虑联合索引

#### 6.4.2 不适合创建索引的场景

- where子句中用不到的字段，最好不要创建索引

- 数据量小的表最好不要创建索引，小于1000行

- 有大量重复数据的列

- 避免对经常更新的表创建过多索引

- 删除不再使用或很少使用的索引

- 不要定义冗余或重复的索引

## 7. EXPLAIN

一些常用的性能参数如下

![](http://img.zqqiliyc.love/mysql/202212071542866.png)

通过

```sql
show status like '变量名'; // 查看指定的变量状态
```

统计SQL查询的执行成本`last_query_cost`。

```sql
slow_query_log=ON; // 开启慢查询日志功能
long_query_time=10; // 设置>=10秒就是慢查询
mysqladmin -u name -p flush-logs slow // 刷新慢查询日志
```

开启`profiling`变量为打开状态`set profiling = on`，后使用`show profiles`和`show profile [for query seq_num]`查看最近的查询资源消耗情况。

EXPLAIN关键字语句输出的各个列的作用说明如下图所示

![](http://img.zqqiliyc.love/mysql/202304260043482.png)

- id如果相同可以认为是一组，从上往下顺序执行

- 在所有组中，id值越大，优先级越高，越先执行

- id号每个号码，表示一趟独立的查询，一个sql的查询趟数越少越好

type字段从好到坏：

**system > const > eq_ref > ref** > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > **range > index > ALL**

### 7.1 EXPLAIN小结😊

+ EXPLAIN不考虑各种Cache。
+ EXPLAIN不能显示MySQL在执行查询时所作的优化工作。
+ EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况。
+ 部分统计信息是估算的，并非精确值。

### 7.2 EXPLAIN的进一步使用

EXPLAIN有四种输出格式，分别是==传统格式，JSON格式，Tree格式以及可视化输出==。

```sql
EXPLAIN format = json|tree select statement;
```

### 7.3 分析优化器的执行计划：trace

`OPTIMIZER_TRACE`可以跟踪优化器所作的各种决策。此功能默认关闭，打开方式如下

```sql
set optimizer_trace = 'enabled=on',end_markers_in_json=on;
set optimizer_trace_max_mem_size=1000000;
```

## 8. 索引优化与查询优化😊

### 8.1 索引失效的场景

- 不满足最左前缀匹配
- where子句中范围条件要放在最后
- != 或 <> 都不可以使用索引，is null 可使用，is not null 不可以使用
- like 以%开头导致索引失效
- OR 前后存在非索引的列，则索引失效

### 8.2  关联查询的优化

+ 左外连接
  + 给被驱动表的用于连接条件的字段添加索引。


+ 内连接
  + 若连接条件中只能有一个字段有索引，则有索引的表充当被驱动表。
  + 小表驱动大表，进一步说是每行数据越小越可能充当驱动表，因为`join_buffer_size`大小确定的情况下，数据行越小，buffer中能装的数据就越多。
  + 为被驱动表匹配的条件增加索引(减少内层表的循环匹配次数)
  + 增大`join_buffer_size`的大小
  + 减少驱动表中不必要字段的查询
  
### 8.3 子查询优化
+ 能直接多表关联的，尽量使用多表关联
+ 不建议使用子查询，建议将子查询结合程序拆开或进行join查询

### 8.4 排序优化
+ where子句条件加索引可避免全表扫描。
+ 在order by 子句加索引避免`filesort`排序。但有些情况下出现`filesort`不一定就性能低。
+ 尽量使用index完成order by排序。如果where和order by后面是相同的列就使用单列索引，否则考虑建立联合索引。
+ 无法使用index时，对`filesort`方式进行优化。
  + filesort优化思路：
  + 尝试提高`sort_buffer_size`的大小
  + 尝试提高`max_length_for_sort_data`的大小
  + `Order By`时`select *` 是大忌，最好只查询需要的字段即可

### 8.5 Group By优化

![](http://img.zqqiliyc.love/mysql/202212091517626.png)

### 8.6 优化分页查询

一般分页查询时，通过创建覆盖索引能过获得更好的性能。一个常见又非常头疼的问题就是`limit 2000000,10`，此时MySQL需要排序前2000000条数据，然后只取之后的10条数据，查询时排序的代价很大。

+ 优化思路1：在索引上完成排序操作，最后根据主键关联回表查询

```sql
SELECT * FROM stu s , (SELECT id FROM stu ORDER BY id LIMIT 2000000,10) tmp WHERE s.id = tmp.id
```

+ 优化思路2：该方案适合主键自增的场景，可把limit查询转化成具体的某个位置的查询

```sql
SELECT * FROM stu s WHERE id > 2000000 LIMIT 10;
```

## 9. 事务

**ACID**，分别是：

- **原子性(Atomicity)**，指事务是一个不可分割的单位，要么都成功，要么都失败。Wiki：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。

- **一致性(Consistency)**，是[数据库系统](https://zh.m.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F "数据库系统")的一项要求：任何数据库事务修改数据必须满足定义好的规则，包括[数据完整性](https://zh.m.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%AE%8C%E6%95%B4%E6%80%A7 "数据完整性")（约束）、[级联回滚](https://zh.m.wikipedia.org/wiki/%E7%BA%A7%E8%81%94%E5%9B%9E%E6%BB%9A "级联回滚")、[触发器](https://zh.m.wikipedia.org/wiki/%E8%A7%A6%E5%8F%91%E5%99%A8_(%E6%95%B0%E6%8D%AE%E5%BA%93) "触发器 (数据库)")等。

- **隔离性(Isolation)**，定义了[数据库](https://zh.m.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93 "数据库")系统中一个事务中操作的结果在何时以何种方式对其他[并发](https://zh.m.wikipedia.org/wiki/%E5%B9%B6%E5%8F%91 "并发")事务操作可见。

- **持久性（Durability）**，已被提交的事务对数据库的修改应该永久保存在数据库中。

总结：ACID是事务的四大特性，在这四个特性中，原子性是基础，隔离性是手段，一致性是约束条件，持久性是最终目的。
数据库事务，本质上就是数据库设计者为了方便起见，把需要保证ACID的一个或者多个数据库操作称为一个事务。

### 9.1 数据并发问题

#### 9.1.1 隔离级别

事务的并发问题(严重性从高到低)：

- **脏写**，指事务A修改了事务B**未提交的数据**，则发生了脏写。
- **脏读**，指事务A**读取到了**事务B已经修改但是**还未提交的数据**，即事务A读取到了临时数据。
- **不可重复读**，事务A**读取**了某个字段，事务B了**更新**了那个字段，之后事务A**读取到了不同值**，若事务B又更新了该字段，那么事务A又可能读取到不同的值。
- **幻读**，事务A读取到了一些字段，之后事务B添加了几行，事务A再次读取时发现**记录数增加**。

MySQL支持的4种**隔离级别**：

- **READ UNCOMMITTED**，解决了脏写。
- **READ COMMITTED**，解决了脏写、脏读。
- **REPEATABLE READ**，解决脏写、脏读和不可重复读。
- **SERIALIZABLE**，串行化，最安全同时也是性能最差。

#### 9.1.2 MySQL的事务日志

事务的四种特性中：

- 隔离性是由锁机制解决的
- 其他三种是通过Redo和Undo日志解决的
  - **Redo日志**，重做日志，提供再写入操作，恢复提交事务的修改的页操作，用来保证事务的持久性。
  - **Undo日志**，回滚日志，回滚行记录到某个版本，保证原子性和一致性。

+ REDO和UNDO都可以视为是一种“恢复”操作。

- Redo Log，由InnoDB生成，记录的是“物理级别”上的页修改操作，比如页号，偏移量，数据的变化
- Undo Log，由InnoDB生成，记录的是“逻辑操作”，记录每个修改操作的**逆操作**，主要用于事务的回滚和一致性非锁定读（MVCC）。

#### 9.1.3 Redo日志

InnoDB引擎事务采用的是WAL(Write-Ahead logging)技术，思想就是先写日志，再写磁盘，只有日志写成功，才算事务提交成功，这里的日志就是redo日志。当服务器宕机时，就算内存中的数据未同步到磁盘，当服务器重启后，通过redo log来恢复，保证了持久性。

1. Redo日志的组成

   - 内存层面，MySQL服务器启动时，会申请一片内存空间，称为redo log buffer，可通过系统变量Innodb_log_buffer_size查看该缓冲区的大小（默认16MB）。这片连续的内存空间又被划分为若干连续的redo log block，默认每个512B。
   - 磁盘层面，redo log file，是持久化的。

2. Redo log的刷盘过程

   ![](http://img.zqqiliyc.love/mysql/202304260043736.png)

3. Redo log的刷新策略，系统变量Innodb_flush_log_at_trx_commit，可选值：

   - 0，每次事务提交时，不进行刷盘操作。(系统默认master thread 每隔1s同步一次)
   - 1，每次提交时，都会进行刷盘（默认值）
   - 2，每次事务提交时，都会把redo log buffer 中的内容提交到page cache，不进行同步，由os决定何时同步到磁盘

4. redo日志的优点：
   1. 降低了刷盘频率
   2. redo日志占用的空间非常小
   3. redo日志是顺序写入磁盘的
   4. 事务执行过程中，redo日志会不断记录。
   5. redo log和bin log的区别，redo log是`存储引擎层`实现的，而bin log是`数据库层`实现的。假设一个事务对某张表插入了10万行数据，则redo log就会在执行了顺序地记录10条日志，而bin log不会，知道事务提交，bin log才会记录这次更改写入到bin log文件中。

#### 9.1.4 Undo日志

redo log是事务持久性的保证，undo log是`事务原子性的保证`，在事务中`更新数据`的前置操作是`先写undo log日志`。undo log是“逻辑日志”，当数据在回滚时，只是逻辑上回滚，但是物理磁盘上数据的结构可能大相径庭。

此外，undo log会`产生redo log`，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。

作用：

+ **回滚数据**，即逻辑回滚。
+ **MVCC**，即多版本并发控制。

## 9.2 锁机制

事务的隔离性由锁机制来实现。

### 9.2.1 并发问题的解决方案

- **方案一**：读操作利用多版本并发控制，写操作加锁。
  ![](http://img.zqqiliyc.love/mysql/202304260044260.png)

+ **方案二**：读写操作均加锁方式。

![](http://img.zqqiliyc.love/mysql/202212112047843.png)

### 9.2.2 锁的分类

#### 9.2.2.1 从数据操作层面看

+ 读锁/共享锁(Shared Lock)
+ 写锁/独占锁(Exclusive Lock)

#### 9.2.2.2 从锁的粒度看

+ 表级锁
+ 行级锁
+ 列级锁

#### 9.2.2.3 加锁方式

+ 显示锁
+ 隐式锁
