## 1. 用户管理及权限管理

命令行登陆参数

```sql
mysql -h [hostname|hostip] -P [port] -u [username] -p [password] [schema] -e [sql-statement]
```

### 1.1 创建用户

核心数据库**mysql**,中的user表记录了所有的用户数据。

创建用户：

- create user 'username'@'hostname' identified by 'password';

修改用户:

- update mysql.use set use = '...' where user and host; flush privileges;

删除用户:

- drop user user@host; or delete from user where user = '...' and host = '..';

修改密码:

- 设置当前用户自己的密码
  - alter mysql.user user() identified by 'password';
  - set password='....';
- 设置别人的密码
  - alter mysql.user 'user'@'host' identified by 'password';
  - set password for 'user'@'host' = 'password';

手动设置密码过期：

- alter user 'user'@'host' password expire;
  用户仍然可以登陆数据库，但是不能进行其他操作，除非重新设置了新密码。
  mysql使用全局变量 default_password_lifetime 默认是0，表示禁用密码过期；可以是正整数，表示N天后密码过期。

### 1.2 权限管理

查看用户权限:    show grants [for 'user'@'host'];

#### 1.2.1 直接赋予权限

- grant 权限1,权限2... on 数据库名.表名 to 'user'@'host' [identified by 'pwd'];

#### 1.2.2 收回权限

- revoke 权限1,权限2... on 数据库名.表名 from 'user'@'host';

#### 1.2.3 如何管理的

mysql是用一张权限表来控制权限的。mysql数据库，user表，db表，table_priv,column_privi,proc_privi。

#### 1.2.4 角色的创建和使用

角色是对权限的“封装”，将角色赋予给具体的用户。

- create role 'rolename'@'host';
- grant privileges on schema to role;

对角色的crud与角色的语法大致相同。

将角色的权限赋予给用户：

- grant role to user;

当角色赋予后，需要**激活**，

- set default role all to user1,user2...;

## 2. Mysql的逻辑架构

- 连接管理-连接层
- 解析优化-服务层
- 存储引擎-引擎层

mysql是一个C/S架构。

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

![](https://raw.githubusercontent.com/Nottoocold/img/main/mysql/mysql%E9%A1%B5%E7%BB%93%E6%9E%84.png)

- File_Header又可被细分如下属性：

![](https://raw.githubusercontent.com/Nottoocold/img/main/mysql/file-header.png)

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

- alter table add **[fulltext|unique] index** idx_name (col_name(len));
- create index **[fulltext|unique]** idx_name on **table_name**(col_name); 

### 6.3 删除索引

- alter table table_name drop index idx_name;
- drop index idx_name on table_name;

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

EXPLAIN关键字语句输出的各个列的作用说明如下图所示

![](https://raw.githubusercontent.com/Nottoocold/img/main/mysql/explain.png)

- id如果相同可以认为是一组，从上往下顺序执行

- 在所有组中，id值越大，优先级越高，越先执行

- id号每个号码，表示一趟独立的查询，一个sql的查询趟数越少越好

type字段从好到坏：

**system > const > eq_ref > ref** > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > **range > index > ALL**

## 8. 索引优化与查询优化

### 8.1 索引失效的场景

- 不满足最左前缀匹配
- where字句中范围条件要放在最后
- != 或 <> 都不可以使用索引，is null 可使用，is not null 不可以使用
- like 以%开头导致索引失效
- OR 前后存在非索引的列，则索引失效



## 9. 事务

ACID，分别是原子性(Atomicity)，一致性(Consistency)，隔离性(Isolation)，
