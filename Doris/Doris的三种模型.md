1. Aggregation聚合类型
	 假设业务有如下数据表模式。
```sql
	 CREATE TABLE IF NOT EXISTS test_db.example_site_visit
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
	`last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
AGGREGATE KEY(`user_id`, `date`, `city`, `age`, `sex`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 10
properties(
"replication_num"="1"
);
```
- 根据user_id, date, city, age, sex这5个key进行数据的聚合
- replace表示取最后一个insert的数据；但在同一个insert中如果包含多条数据，会随机取一条
- 1. 数据insert时，会对同一个insert批次的数据进行聚合2. BE进行Compaction时，会对不同insert批次的数据进行聚合3. 用户进行查询时，在BE后端可能不同insert批次的数据未进行聚合，此时会对符合查询条件的数据进行内部聚合(**不用用户调用group by，会扫描所有列的数据**)后，再返回给客户端
- 所有的key列必须在value列之前
2. Unique模型
   ```sql
   CREATE TABLE IF NOT EXISTS test_db.user
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `phone` LARGEINT COMMENT "用户电话",
    `address` VARCHAR(500) COMMENT "用户地址",
    `register_time` DATETIME COMMENT "用户注册时间"
)
UNIQUE KEY(`user_id`, `username`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 10
properties(
"replication_num"="1"
)
```
- Unique其实是Aggregate数据模型的一种特例
- 根据user_id, username这2个key进行数据的聚合，其余字段按replace方式进行聚合
3. Duplicate模型
	```sql
	CREATE TABLE IF NOT EXISTS test_db.example_log
(
    `timestamp` DATETIME NOT NULL COMMENT "日志时间",
    `type` INT NOT NULL COMMENT "日志类型",
    `error_code` INT COMMENT "错误码",
    `error_msg` VARCHAR(1024) COMMENT "错误详细信息",
    `op_id` BIGINT COMMENT "负责人id",
    `op_time` DATETIME COMMENT "处理时间"
)
DUPLICATE KEY(`timestamp`, `type`)
DISTRIBUTED BY HASH(`timestamp`) BUCKETS 10
properties(
"replication_num"="1"
);
```
   - 数据不会发生内部聚合，插入多少条数据，查询就会返回多少条数据
   - duplicate key只是指定了timestamp和type两个sorted column, 用于数据排序，并不能作为数据唯一的标识
   ### 列定义
   AGGREGATE KEY数据模型中，所有没有指定聚合方式（SUM、REPLACE、MAX、MIN）的列视为Key列。而其余则为Value列
### ENGINE
   Doris 支持的引擎有：**olap|mysql|broker|hive**
   只有Olap是Doris自己管理的，其他都是相当于映射关系
   ### 分区和分桶
   1）Partition列可以指定一列或多列。分区列必须为KEY列。多列分区的使用方式在后面介绍。
   2）范围分区Rang 分区列通常为**时间列**（ PARTITION BY RANGE(`date`) ），以方便的管理新旧数据
   3）List **分区（列表分区）**
   **只有当数据为目标分区枚举值其中之一时，才可以命中分区**
   4）**分桶(Bucket)**
   Ø 如果**使用了 Partition**，则 DISTRIBUTED ... 语句描述的是数据在**各个分区内**的划分规则。如果**不使用 Partition**，则描述的是对**整个表**的数据的划分规则。
   5)**复合分区与单分区**
   **复合分区**：分区和分桶
   **单分区：只分桶（其实是所有数据在一个分区，数据只做 hash 分布）
   6)多列分区
### PROPERTIES
- replication_num
	  副本数。默认副本数为3，如果BE节点数量小于3，则需要指副本数小于等于BE节点数量
- storage_medium/storage_cooldown_time
		  storage_medium用于声明表数据的初始存储介质。而 storage_cooldown_time 用于设定到期时间。
		  示例：
		`"storage_medium" = "SSD",`  
		`"storage_cooldown_time" = "2020-11-20 00:00:00"`
		这个示例表示数据存放在 SSD 中，并且在 2020-11-20 00:00:00 到期后，会自动迁移到 HDD 存储上。
### 动态分区
目前实现了**动态添加**分区及**动态删除**分区的功能。
动态分区只**支持 Range 分区**。
```sql
create table student_dynamic_partition1
(
id int,
time date,
name varchar(50),
age int
)
duplicate key(id,time)
PARTITION BY RANGE(time)()
distributed by hash(`id`)
PROPERTIES(
"dynamic_partition.enable" = "true",
"dynamic_partition.time_unit" = "DAY",
"dynamic_partition.create_history_partition" = "true",
"dynamic_partition.history_partition_num" = "3",
"dynamic_partition.start" = "-7",
"dynamic_partition.end" = "3",
"dynamic_partition.prefix" = "p",
"dynamic_partition.buckets" = "10",
 "replication_num" = "1"
 );
```
