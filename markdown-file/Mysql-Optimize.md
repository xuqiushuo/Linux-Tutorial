# MySQL 优化


- 下面说的优化基于 MySQL 5.6，理论上 5.5 之后的都算适用，具体还是要看官网

## 优秀材料

- <https://notes.diguage.com/mysql/>
- <>
- <>
- <>
- <>


## 服务状态查询

- 查看当前数据库的状态，常用的有：
	- 查看系统状态：`SHOW STATUS;`
	- 查看刚刚执行 SQL 是否有警告信息：`SHOW WARNINGS;`
	- 查看刚刚执行 SQL 是否有错误信息：`SHOW ERRORS;`
	- 查看已经连接的所有线程状况：`SHOW FULL PROCESSLIST;`
	    - 输出参数说明：<http://www.ibloger.net/article/2519.html>
	    - 可以结束某些连接：`kill id值`
	- 查看当前连接数量：`SHOW STATUS LIKE 'max_used_connections';`
	- 查看变量，在 my.cnf 中配置的变量会在这里显示：`SHOW VARIABLES;`
	- 查询慢 SQL 配置：`show variables like 'slow%';`
	    - 开启慢 SQL：`set global slow_query_log='ON'`
    - 查询慢 SQL 秒数值：` show variables like 'long%';`
        - 调整秒速值：`set long_query_time=1;`
	- 查看当前MySQL 中已经记录了多少条慢查询，前提是配置文件中开启慢查询记录了.
		- `SHOW STATUS LIKE '%slow_queries%';`
	- 查询当前MySQL中查询、更新、删除执行多少条了，可以通过这个来判断系统是侧重于读还是侧重于写，如果是写要考虑使用读写分离。
		- `SHOW STATUS LIKE '%Com_select%';`
		- `SHOW STATUS LIKE '%Com_insert%';`
		- `SHOW STATUS LIKE '%Com_update%';`
		- `SHOW STATUS LIKE '%Com_delete%';`
	- 如果 rollback 过多，说明程序肯定哪里存在问题
		- `SHOW STATUS LIKE '%Com_rollback%';`
	- 显示MySQL服务启动运行了多少时间，如果MySQL服务重启，该时间重新计算，单位秒
		- `SHOW STATUS LIKE 'uptime';`
	- 显示查询缓存的状态情况
		- `SHOW STATUS LIKE 'qcache%';`
		- PS.下面的解释，我目前不肯定是对，还要再找下资料：
			- <http://dba.stackexchange.com/questions/33811/qcache-free-memory-not-full-yet-i-get-alot-of-qcache-lowmem-prunes>
			- <https://dev.mysql.com/doc/refman/5.7/en/query-cache-status-and-maintenance.html>
			- <https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html>
			- <http://www.111cn.net/database/110/c0c88da67b9e0c6c8fabfbcd6c733523.htm>
		- 1. Qcache_free_blocks，缓存中相邻内存块的个数。数目大说明可能有碎片。如果数目比较大，可以执行： `flush query cache;` 对缓存中的碎片进行整理，从而得到一个空闲块。 
		- 2. Qcache_free_memory，缓存中的空闲内存大小。如果 Qcache_free_blocks 比较大，说明碎片严重。 如果 Qcache_free_memory 很小，说明缓存不够用了。 
		- 3. Qcache_hits，每次查询在缓存中命中时就增大该值。 
		- 4. Qcache_inserts，每次查询，如果没有从缓存中找到数据，这里会增大该值
		- 5. Qcache_lowmem_prunes，因内存不足删除缓存次数，缓存出现内存不足并且必须要进行清理, 以便为更多查询提供空间的次数。返个数字最好长时间来看；如果返个数字在不断增长，就表示可能碎片非常严重，或者缓存内存很少。 
		- 6. Qcache_not_cached # 没有进行缓存的查询的数量，通常是这些查询未被缓存或其类型不允许被缓存
		- 7. Qcache_queries_in_cache # 在当前缓存的查询（和响应）的数量。 
		- 8. Qcache_total_blocks #缓存中块的数量。
- 查询哪些表在被使用，是否有锁表：`SHOW OPEN TABLES WHERE In_use > 0;`
- 查询 innodb 状态（输出内容很多）：`SHOW ENGINE INNODB STATUS;`
- 锁性能状态：`SHOW STATUS LIKE  'innodb_row_lock_%';`
    - Innodb_row_lock_current_waits：当前等待锁的数量
    - Innodb_row_lock_time：系统启动到现在、锁定的总时间长度
    - Innodb_row_lock_time_avg：每次平均锁定的时间
    - Innodb_row_lock_time_max：最长一次锁定时间
    - Innodb_row_lock_waits：系统启动到现在、总共锁定次数
- 帮我们分析表，并提出建议：`select * from my_table procedure analyse();`


## 系统表

- 当前运行的所有事务：`select * from information_schema.INNODB_TRX;`
- 当前事务出现的锁：`select * from information_schema.INNODB_LOCKS;`
- 锁等待的对应关系：`select * from information_schema.INNODB_LOCK_WAITS;`


## 查询优化

- 使用 EXPLAIN 进行 SQL 语句分析：`EXPLAIN SELECT * FROM sys_user;`
- 得到的结果有下面几列：
	- **id**，该列表示当前结果序号，无特殊意义，不重要
	- **select_type**，表示 SELECT 语句的类型，有下面几种
		- SIMPLE，表示简单查询，其中不包括连接查询和子查询
		- PRIMARY，表示主查询，或者是最外面的查询语句。比如你使用一个子查询语句，比如这条 SQL：`EXPLAIN SELECT * FROM (SELECT sys_user_id FROM sys_user WHERE sys_user_id = 1) AS temp_table;`
			- 这条 SQL 有两个结果，其中有一个结果的类型就是 PRIMARY
		- UNION，使用 UNION 的 SQL 是这个类型
		- DERIVED，在 SQL 中 From 后面子查询
		- SUBQUERY，子查询
		- 还有其他一些
	- **table**，表名或者是子查询的一个结果集
	- **type**，表示表的链接类型，分别有（以下的连接类型的顺序是从最佳类型到最差类型）**（这个属性重要）**：
		- 性能好：
			- system，表仅有一行，这是 const 类型的特列，平时不会出现，这个也可以忽略不计。
			- const，数据表最多只有一个匹配行，因为只匹配一行数据，所以很快，常用于 PRIMARY KEY 或者 UNIQUE 索引的查询，可理解为 const 是最优化的。
			- eq_ref，mysql 手册是这样说的:"对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了 const 类型。它用在一个索引的所有部分被联接使用并且索引是 UNIQUE(唯一键) 也不是 PRIMARY KEY(主键)"。eq_ref 可以用于使用 = 比较带索引的列。
			- ref，查询条件索引既不是 UNIQUE(唯一键) 也不是 PRIMARY KEY(主键) 的情况。ref 可用于 = 或 < 或 > 操作符的带索引的列。
			- ref_or_null，该联接类型如同 ref，但是添加了 MySQL 可以专门搜索包含 NULL 值的行。在解决子查询中经常使用该联接类型的优化。
		- 性能较差：
			- index_merge，该联接类型表示使用了索引合并优化方法。在这种情况下，key 列包含了使用的索引的清单，key_len 包含了使用的索引的最长的关键元素。
			- unique_subquery，该类型替换了下面形式的IN子查询的ref: `value IN (SELECT primary_key FROM single_table WHERE some_expr)`。unique_subquery 是一个索引查找函数,可以完全替换子查询,效率更高。
			- index_subquery，该联接类型类似于 unique_subquery。可以替换 IN 子查询, 但只适合下列形式的子查询中的非唯一索引: `value IN (SELECT key_column FROM single_table WHERE some_expr)`
			- range，只检索给定范围的行, 使用一个索引来选择行。
			- index，该联接类型与 ALL 相同, 除了只有索引树被扫描。这通常比 ALL 快, 因为索引文件通常比数据文件小。
		- 性能最差：
			- ALL，对于每个来自于先前的表的行组合, 进行完整的表扫描。（性能最差）
	- **possible_keys**，指出 MySQL 能使用哪个索引在该表中找到行。如果该列为 NULL，说明没有使用索引，可以对该列创建索引来提供性能。**（这个属性重要）**
	- **key**，显示 MySQL 实际决定使用的键 (索引)。如果没有选择索引, 键是 NULL。**（这个属性重要）**
	- **key**_len，显示 MySQL 决定使用的键长度。如果键是 NULL, 则长度为 NULL。注意：key_len 是确定了 MySQL 将实际使用的索引长度。
	- **ref**，显示使用哪个列或常数与 key 一起从表中选择行。
	- **rows**，显示 MySQL 认为它执行查询时必须检查的行数。**（这个属性重要）**
	- **Extra**，该列包含 MySQL 解决查询的详细信息：
		- Distinct:MySQL 发现第 1 个匹配行后, 停止为当前的行组合搜索更多的行。
		- Not exists:MySQL 能够对查询进行 LEFT JOIN 优化, 发现 1 个匹配 LEFT JOIN 标准的行后, 不再为前面的的行组合在该表内检查更多的行。
		- range checked for each record (index map: #):MySQL 没有发现好的可以使用的索引, 但发现如果来自前面的表的列值已知, 可能部分索引可以使用。
		- Using filesort:MySQL 需要额外的一次传递, 以找出如何按排序顺序检索行。
		- Using index: 从只使用索引树中的信息而不需要进一步搜索读取实际的行来检索表中的列信息。
		- Using temporary: 为了解决查询,MySQL 需要创建一个临时表来容纳结果。
		- Using where:WHERE 子句用于限制哪一个行匹配下一个表或发送到客户。
		- Using sort_union(...), Using union(...), Using intersect(...): 这些函数说明如何为 index_merge 联接类型合并索引扫描。
		- Using index for group-by: 类似于访问表的 Using index 方式,Using index for group-by 表示 MySQL 发现了一个索引, 可以用来查 询 GROUP BY 或 DISTINCT 查询的所有列, 而不要额外搜索硬盘访问实际的表。
- **了解对索引不生效的查询情况 （这个属性重要）**
	- 使用 LIKE 关键字的查询，在使用 LIKE 关键字进行查询的查询语句中，如果匹配字符串的第一个字符为“%”，索引不起作用。只有“%”不在第一个位置，索引才会生效。
	- 使用联合索引的查询，MySQL 可以为多个字段创建索引，一个索引可以包括 16 个字段。对于联合索引，只有查询条件中使用了这些字段中第一个字段时，索引才会生效。
	- 使用 OR 关键字的查询，查询语句的查询条件中只有 OR 关键字，且 OR 前后的两个条件中的列都是索引列时，索引才会生效，否则，索引不生效。
- 子查询优化
	- MySQL 从 4.1 版本开始支持子查询，使用子查询进行 SELECT 语句嵌套查询，可以一次完成很多逻辑上需要多个步骤才能完成的 SQL 操作。
	- 子查询虽然很灵活，但是执行效率并不高。
	- 执行子查询时，MYSQL 需要创建临时表，查询完毕后再删除这些临时表，所以，子查询的速度会受到一定的影响。
	- 优化：
		- 可以使用连接查询（JOIN）代替子查询，连接查询时不需要建立临时表，其速度比子查询快。

## 查询不走索引优化

- WHERE字句的查询条件里有不等于号（WHERE column!=…），MYSQL将无法使用索引
- 类似地，如果WHERE字句的查询条件里使用了函数（如：WHERE DAY(column)=…），MYSQL将无法使用索引
- 在JOIN操作中（需要从多个数据表提取数据时），MYSQL只有在主键和外键的数据类型相同时才能使用索引，否则即使建立了索引也不会使用
- 如果WHERE子句的查询条件里使用了比较操作符LIKE和REGEXP，MYSQL只有在搜索模板的第一个字符不是通配符的情况下才能使用索引。比如说，如果查询条件是LIKE 'abc%',MYSQL将使用索引；如果条件是LIKE '%abc'，MYSQL将不使用索引。
- 在ORDER BY操作中，MYSQL只有在排序条件不是一个查询条件表达式的情况下才使用索引。尽管如此，在涉及多个数据表的查询里，即使有索引可用，那些索引在加快ORDER BY操作方面也没什么作用。
- 如果某个数据列里包含着许多重复的值，就算为它建立了索引也不会有很好的效果。比如说，如果某个数据列里包含了净是些诸如“0/1”或“Y/N”等值，就没有必要为它创建一个索引。
- 索引有用的情况下就太多了。基本只要建立了索引，除了上面提到的索引不会使用的情况下之外，其他情况只要是使用在WHERE条件里，ORDER BY 字段，联表字段，一般都是有效的。 建立索引要的就是有效果。 不然还用它干吗？ 如果不能确定在某个字段上建立的索引是否有效果，只要实际进行测试下比较下执行时间就知道。
- 如果条件中有or(并且其中有or的条件是不带索引的)，即使其中有条件带索引也不会使用(这也是为什么尽量少用or的原因)。注意：要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引
- 如果列类型是字符串，那一定要在条件中将数据使用引号引用起来,否则不使用索引
- 如果mysql估计使用全表扫描要比使用索引快,则不使用索引


## 其他查询优化

- 关联查询过程
    - 确保 ON 或者 using子句中的列上有索引
    - 确保任何的 groupby 和 orderby 中的表达式只涉及到一个表中的列。
- count()函数优化
    - count()函数有一点需要特别注意：它是不统计值为NULL的字段的！所以：不能指定查询结果的某一列，来统计结果行数。即 count(xx column) 不太好。
    - 如果想要统计结果集，就使用 count(*)，性能也会很好。
- 分页查询（数据偏移量大的场景）
    - 不允许跳页，只能上一页或者下一页
    - 使用 where 加上上一页 ID 作为条件(具体要看 explain 分析效果)：`select xxx,xxx from test_table where id < '上页id分界值' order by id desc limit 20;`

## 创表原则

- 所有字段均定义为 NOT NULL ，除非你真的想存 Null。因为表内默认值 Null 过多会影响优化器选择执行计划


## 建立索引原则

- 使用区分度高的列作为索引，字段不重复的比例，区分度越高，索引树的分叉也就越多，一次性找到的概率也就越高。
- 尽量使用字段长度小的列作为索引
- 使用数据类型简单的列（int 型，固定长度）
- 选用 NOT NULL 的列。在MySQL中，含有空值的列很难进行查询优化，因为它们使得索引、索引的统计信息以及比较运算更加复杂。你应该用0、一个特殊的值或者一个空串代替空值。
- 尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。这样也可避免索引重复。



## 数据库结构优化

- 将字段很多的表分解成多个表
	- 对于字段较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。
	- 因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。
- 增加中间表
	- 对于需要经常联合查询的表，可以建立中间表以提高查询效率。
	- 通过建立中间表，将需要通过联合查询的数据插入到中间表中，然后将原来的联合查询改为对中间表的查询。
- 增加冗余字段
	- 设计数据表时应尽量遵循范式理论的规约，尽可能的减少冗余字段，让数据库设计看起来精致、优雅。但是，合理的加入冗余字段可以提高查询速度。



## 插入数据的优化（适用于 InnoDB）

- 插入数据时，影响插入速度的主要是索引、唯一性校验、一次插入的数据条数等。
- 开发环境情况下的考虑：
	- 开发场景中，如果需要初始化数据，导入数据等一些操作，而且是开发人员进行处理的，可以考虑在插入数据之前，先禁用整张表的索引，
		- 禁用索引使用 SQL：`ALTER TABLE table_name DISABLE KEYS;`
		- 当导入完数据之后，重新让MySQL创建索引，并开启索引：`ALTER TABLE table_name ENABLE KEYS;`
	- 如果表中有字段是有唯一性约束的，可以先禁用，然后在开启：
		- 禁用唯一性检查的语句：`SET UNIQUE_CHECKS = 0;`
		- 开启唯一性检查的语句：`SET UNIQUE_CHECKS = 1;`
	- 禁用外键检查（建议还是少量用外键，而是采用代码逻辑来处理）
		- 插入数据之前执行禁止对外键的检查，数据插入完成后再恢复，可以提供插入速度。
		- 禁用：`SET foreign_key_checks = 0;`
		- 开启：`SET foreign_key_checks = 1;`
	- 使用批量插入数据
	- 禁止自动提交
		- 插入数据之前执行禁止事务的自动提交，数据插入完成后再恢复，可以提供插入速度。
		- 禁用：`SET autocommit = 0;`
		- 开启：`SET autocommit = 1;`



## 服务器优化

- 好硬件大家都知道，这里没啥好说，如果是 MySQL 单独一台机子，那机子内存可以考虑分配 60%~70% 给 MySQL
- 通过优化 MySQL 的参数可以提高资源利用率，从而达到提高 MySQL 服务器性能的目的。
	- 可以看我整理的这篇文章：<https://github.com/judasn/Linux-Tutorial/blob/master/MySQL-Settings/MySQL-5.6/1G-Memory-Machine/my-for-comprehensive.cnf>
- 由于 binlog 日志的读写频繁，可以考虑在 my.cnf 中配置，指定这个 binlog 日志到一个 SSD 硬盘上。


## 锁相关

InnoDB支持事务；InnoDB 采用了行级锁。也就是你需要修改哪行，就可以只锁定哪行。
在 Mysql 中，行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql 语句操作了主键索引，Mysql 就会锁定这条主键索引；如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引。
InnoDB 行锁是通过给索引项加锁实现的，如果没有索引，InnoDB 会通过隐藏的聚簇索引来对记录加锁。也就是说：如果不通过索引条件检索数据，那么InnoDB将对表中所有数据加锁，实际效果跟表锁一样。因为没有了索引，找到某一条记录就得扫描全表，要扫描全表，就得锁定表。


数据库的增删改操作默认都会加排他锁，而查询不会加任何锁。

排他锁：对某一资源加排他锁，自身可以进行增删改查，其他人无法进行任何操作。语法为：
select * from table for update;

共享锁：对某一资源加共享锁，自身可以读该资源，其他人也可以读该资源（也可以再继续加共享锁，即 共享锁可多个共存），但无法修改。
要想修改就必须等所有共享锁都释放完之后。语法为：
select * from table lock in share mode;



## 资料

- <https://my.oschina.net/jsan/blog/653697>
- <https://blog.imdst.com/mysql-5-6-pei-zhi-you-hua/>
- <https://mp.weixin.qq.com/s/qCRfxIr1RoHd9i8-Hk8iuQ>
- <https://yancg.cn/detail?id=3>
- <https://www.jianshu.com/p/1ab3cd5551b9>
- <http://blog.brucefeng.info/post/mysql-index-query?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io>
- <>
- <>
- <>
- <>
- <>
- <>