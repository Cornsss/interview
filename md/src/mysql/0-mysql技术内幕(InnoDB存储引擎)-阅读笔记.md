# **Chapter02. InnoDB的介绍**

![image-20221115153929612](/Users/tracymc_zhu/Library/Application Support/typora-user-images/image-20221115153929612.png)

```mysql
-- InnoDB的缓冲池介绍

-- 查看InnoDB缓冲池实例个数
show variables like '%innodb_buffer_pool_instance%';

-- 查看缓冲池大小：系统设置为128G
show variables like '%innodb_buffer_pool_size%'\G

-- 查看LRU算法插入位置，查到的数量表示插入尾端的37%位置：因为如果某一次查到的数据过于庞大的话，会把前面比较热点的数据替换掉
show variables like '%innodb_old_blocks_pct%'\G		

-- 查看脏页的数量
show engine innodb status\G

----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992 ----> 缓冲池总大小
Dictionary memory allocated 5781067
Buffer pool size   8191 ----> 缓冲池使用大小
Free buffers       1024 ----> 空闲页数量
Database pages     7134	----> LRU总数
Old database pages 2613 ----> LRU的old部分：较少被访问的
Modified db pages  0	----> 脏页数量
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0 --> Pending writes flush list: checkpoint时刷新的脏页数量
Pages made young 2071, not young 24646 ----> LRU old转移到new的页数量
0.00 youngs/s, 0.00 non-youngs/s
Pages read 8211, created 543, written 1620398
0.00 reads/s, 0.00 creates/s, 9.31 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000	----> 命中率为100%，是很好的状态
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7134, unzip_LRU len: 0
I/O sum[509]:cur[0], unzip sum[0]:cur[0]

-- 查看脏页数量 5.7表不存在
select * from innodb_buffer_page_lru where oldest_modification > 0;

-- 查看重做日志缓存配置大小Redo log：系统设置16G
show variables like '%innodb_log_buffer_size%'\G

-- 1. 查看InnoDB的具体运行状态信息
show engine innodb status;
   
-- 2. 查看设置的LRU列表页中可用的数量
show variables like 'innodb_lru_scan_depth'\G

-- 3. 查看强制checkpoint的脏页比例
show variables like 'innodb_max_dirty_pages_pct'\G

-- 4. 设置回收undo页的数量
show variables like '%innodb_purge_batch_size%'\G
```



## 2.0 CheckPoint技术 WAL

**Sharp CheckPoint**
数据库发生关闭时，强制将所有脏页刷新到磁盘
**Fuzzy CheckPoint**
(1) Master Thread 定时按比例刷新脏页
(2) Flush_LRU_list 为了保证LRU列表有足够的空闲页可以使用，移除尾端数据时刷新脏页

```mysql
-- 查看LRU可用页的大小设置
show variables like '%innodb_lru_scan_depth%'\G
```

(3) Async/Sync Flush： 当重做日志文件不可用时，
(4) Dirty_Page_Too_Much 脏页过多



## 2.1 1.0版本之前的Master Thread伪代码

```java
组成：（loop、flush loop、suspend loop、backgroud loop）

	// 1s的任务： 5次、100个属于hard coding
    private static void task1S() throws InterruptedException {
        // 每1s的任务
        for (int i =0; i < 10; i++) {
            Thread.sleep(1000);
            System.out.println("1.1 刷新日志缓存到磁盘中");
            last_second_io_times = RandomUtil.randomInt(5);
            if (last_second_io_times < 5) {
                System.out.println("1.2 上1s的IO次数大于5次，进行合并插入缓存操作");
            }
            buf_get_modified_ratio_pct = RandomUtil.randomInt(85, 100);
            if (buf_get_modified_ratio_pct > InnoDBParam.innodb_max_dirty_pages_pct) {
                System.out.println("1.3 当前的脏页比例已大于配置的脏页阈值，刷新100个脏页到磁盘");
            }
            System.out.println("后台线程..");
        }
    }

	// 10s的任务： 5个、20页、100页、200 属于hard coding
    private static void task10s() {
        last_10second_io_times = RandomUtil.randomInt(180, 200);
        if (last_10second_io_times < 200) { // 当前系统有足够的磁盘IO能力
            System.out.println("2.1 刷新100个脏页到磁盘");
        }
        System.out.println("2.2 合并插入缓冲（至多5个）");
        System.out.println("2.3 刷新日志缓冲");
        System.out.println("2.4 回收Undo页（至多20页）");
        buf_get_modified_ratio_pct = RandomUtil.randomInt(65, 75);
        if (buf_get_modified_ratio_pct > 70) {
            System.out.println("2.5 当前脏页比例过多，刷新100个脏页");
        } else {
            System.out.println("2.5 刷新脏页比例合理，刷新" + buf_get_modified_ratio_pct*0.1 + "个脏页");
        }

    }
```



## 2.2 1.2版本之前的Master Thread伪代码

```mysql

```



## 2.3 1.2版本的Master Thread伪代码

```mysql

```

## 2.4 InnoDB的特性

1. insert buffer

   ```mysql
   -------------------------------------
   INSERT BUFFER AND ADAPTIVE HASH INDEX
   -------------------------------------
   Ibuf: size 1, free list len 60, seg size 62, 7 merges
   ---> size 1: 代表已经合并的数量
   ---> free list len: 空闲列表的长度
   ---> seg size: insert buffer的大小（62*16K=992K）
   ---> 7 merges: 合并次数，也就是实际读取页的次数
   
   merged operations:
    insert 8, delete mark 0, delete 0
   ---> insert: insert buffer
   ---> delete mark: delete buffer
   ---> delete: purge buffer
   
   discarded operations:    ---> 
    insert 0, delete mark 0, delete 0
   Hash table size 34673, node heap has 3 buffer(s)
   Hash table size 34673, node heap has 2 buffer(s)
   Hash table size 34673, node heap has 2 buffer(s)
   Hash table size 34673, node heap has 3 buffer(s)
   Hash table size 34673, node heap has 8 buffer(s)
   Hash table size 34673, node heap has 7 buffer(s)
   Hash table size 34673, node heap has 4 buffer(s)
   Hash table size 34673, node heap has 4 buffer(s)
   44.89 hash searches/s, 32.69 non-hash searches/s
   
   -- 查看开启哪些缓存：update、delete、insert
   show variables like '%innodb_change_buffering%'\G
   
   -- 查看insert buffer占缓冲池大小：默认25%、最大50%
   show variables like '%innodb_change_buffer_max_size%'\G
   
   
   ```

   

2. double write

   mysql通过双写来达到数据的高可靠性，当数据库在数据页的部分写失效情况时，就可以从doublewrite共享表空间中恢复数据。

```mysql
-- 查看doublewrite的运行状态：大约是64：1的情况
show global status like 'innodb_dblwr%'\G	
*************************** 1. row ***************************
Variable_name: Innodb_dblwr_pages_written	---> 一共写的页数
        Value: 3001845
*************************** 2. row ***************************
Variable_name: Innodb_dblwr_writes   ----> 实际写入的页数
        Value: 106867
2 rows in set (0.00 sec)
```



3. AHI

​	mysql在查询数据的访问模式来建立自适应的hash索引，来提高查询效率，但他只支持等值查询。



```mysql
Hash table size 34673, node heap has 3 buffer(s)
Hash table size 34673, node heap has 2 buffer(s)
Hash table size 34673, node heap has 2 buffer(s)
Hash table size 34673, node heap has 3 buffer(s)
Hash table size 34673, node heap has 8 buffer(s)
Hash table size 34673, node heap has 7 buffer(s)
Hash table size 34673, node heap has 4 buffer(s)
Hash table size 34673, node heap has 4 buffer(s)
44.89 hash searches/s, 32.69 non-hash searches/s
```



# Chapter03 文件

## 3.0 错误日志

```mysql
show variables like 'log_error'\G
主要是对启动、运行和关闭过程的记录
```



## 3.1 慢查询日志

```mysql
-- 查看是否启用慢查询
mysql> show variables like '%slow_query_log'\G
*************************** 1. row ***************************
Variable_name: slow_query_log
        Value: OFF
1 row in set (0.00 sec)


-- 查看慢查询的时间参数
mysql> show variables like 'long_query_time'\G
*************************** 1. row ***************************
Variable_name: long_query_time
        Value: 10.000000
1 row in set (0.01 sec)


-- 查看慢查询输出的方式
mysql> show variables like 'log_output'\G
*************************** 1. row ***************************
Variable_name: log_output
        Value: FILE
1 row in set (0.01 sec)
还可以是TABLE


-- 查询慢查询的结果
-- 当前设置的慢查询时间是10s，所以下面的语句会触发
mysql> select sleep(10);
[root@cdscistor01 mysql]# mysqldumpslow /var/lib/mysql/cdscistor01-slow.log
mysql> select * from mysql.slow_log\G
log表使用的csv引擎
```

## 3.2 binary log

1. binlog作用
2. binlog开启
4. 事务的隔离级别
5. 查看binlog：mysqlbinlog工具查看

```mysql
--- 查看binlog_cache_size设置的大小是否合理，默认大小是32K
mysql> show global status like 'binlog_cache%'\G
*************************** 1. row ***************************
Variable_name: Binlog_cache_disk_use
        Value: 0
*************************** 2. row ***************************
Variable_name: Binlog_cache_use
        Value: 0
2 rows in set (0.00 sec)

---  关于sync_binlog参数的说明：
---  默认值为0，设置为1时，表示不适用操作系统的缓冲来写入日志，这时如果发生commit操作之前写日志后宕机，那么就会存在问题。
---  解决方案：配合innodb_support_xa=1参数解决
---  主从架构中的slave角色，不会主动从master中同步binlog，如果需要同步，则需要设置log_slave_update

```

## 3.3 redolog

1. 定义：mysql在datadir下会有两个默认的表空间文件：ib_logfile0、ib_logfile1,官方叫这两个文件是InnoDB存储引擎的日志文件，他其实就是redolog
2. binlog和redolog的区别（redolog的写入是按照512bytes，即一个扇区的缓存大小写入的，是不需要doublewrite的）

​	（1）binlog是记录数据库的开启、运行状态和关闭的日志

​	（2）binlog记录的是一个事务的逻辑操作日志，而redolog记录的是物理页发生更改的日志

​	（3）写入的情景是不同的，在事务提交前会不断的产生redolog，但只会产生一次binlog

# Chapter04 表

```mysql
-- 如何查看表的主键: 这个方法不适用于多个列组成的主键
select *,_rowid from course;

-- 实例证明，回滚信息是存储在共享表空间的
-- 1. 开启每个表独立表空间
mysql> show variables like 'innodb_file_per_table'\G
*************************** 1. row ***************************
Variable_name: innodb_file_per_table
        Value: ON
1 row in set (0.00 sec)
-- 2. 查看当前表的表空间和共享表空间的大小信息
mysql> system ls -lh /var/lib/mysql/zlzjl/enwords.ibd
-rw-r-----. 1 mysql mysql 16M 7月  15 16:44 /var/lib/mysql/zlzjl/enwords.ibd
mysql> system ls -lh /var/lib/mysql/ibdata*
-rw-r-----. 1 mysql mysql 76M 8月   2 16:26 /var/lib/mysql/ibdata1
-- 3. 进行显示的事务操作
set autocommit=0;
update enwords set translation ='test' where word like 'a%';
-- 4.再查看大小信息

-- 3.行记录的内部结构
mysql> select * from mytest\G
*************************** 1. row ***************************
t1: a
t2: bb
t3: bb
t4: ccc
*************************** 2. row ***************************
t1: d
t2: ee
t3: ee
t4: fff
*************************** 3. row ***************************
t1: d
t2: NULL
t3: NULL
t4: fff
3 rows in set (0.00 sec)

Compact：
0000c070  73 75 70 72 65 6d 75 6d  03 02 01 00 00 00 10 00  |supremum........|
0000c080  2c/00 00 00 02 c4 00/00  00 0d bf bb 01/e7 00 00  |,...............|
0000c090  05 10 01 10/61 62 62 62  62 20 20 20 20 20 20 20  |....abbbb       |
0000c0a0  20 63 63 63/03 02 01 00  00 00 18 00 2b 00 00 00  | ccc........+...|
0000c0b0  02 c4 01 00 00 0d bf bb  10 ef 00 00 01 cf 01 10  |................|
0000c0c0  64 65 65 65 65 20 20 20  20 20 20 20 20 66 66 66  |deeee        fff|
0000c0d0 /03 01/06/00 00 20 ff 98/ 00 00 00 02 c4 02/00 00  |..... ..........|
0000c0e0  0d bf bc 47/ab 00 00 01  6b 01 10/64 66 66 66 00  |...G....k..dfff.|------>这里是ROW=3的数据
可以看出在compact模式下，null不占空间

/*03 02 01*/ 变长字段长度的列表：逆序排列。当前表的变长字段的长度按顺序排列是1，2，3，则逆序即为3、2、1
/*00*/ 	当前行是否有NULL值
/*00 00 10 00 2c*/	Row Header 固定5字节
/*00 00 00 02 c4 00*/	RowID
/*00  00 0d bf bb 01*/ Transation ID
/*e7 00 00 05 10 01 10*/	Roll Pointer
/*61*/ 列1数据
/*62 62*/ 列2数据
/*62  62 20 20 20 20 20 20 20 20*/	列3数据：因为这个字段是定长字段，所以不足的位数需要补齐空格
/*63 63 63*/	列4的数据
-- 以下同理推出row=2的数据和row=3的数据

-- 我们用同样的方式可以去研究Redundant格式，实践得知，Redundant行格式的varchar不占存储空间，但是char需要，因为char是定长的

--
```



# Chapter05 索引和算法





# 参数整理5.7为例

| 模块       | 参数名                         | 作用                                                 |
| ---------- | ------------------------------ | ---------------------------------------------------- |
| 慢查询日志 | log_error                      | 记录mysql的开启、运行和关闭的错误日志                |
|            | slow_query_log                 | 查看慢查询是否打开                                   |
|            | long_query_time                | 慢查询设置的时间                                     |
|            | log_output                     |                                                      |
|            | log_queries_not_using_indexes  |                                                      |
|            | log_throttle_not_using_indexes |                                                      |
| binlog     | datadir                        | 存储日志的文件地址                                   |
|            | max_binlog_size                | 最大存储binlog的单个文件大小                         |
|            | binlog_cache_size              | 为每个会话分配的binlog缓存大小，不宜过大，也不宜过小 |
|            | log_slave_update               |                                                      |
|            | Innodb_support_xa              |                                                      |
|            |                                |                                                      |

