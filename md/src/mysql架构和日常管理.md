<img src="C:\Users\tracy\AppData\Roaming\Typora\typora-user-images\image-20220714094721151.png" alt="image-20220714094721151" style="zoom:67%;" />

**查看优化**： set optimizer_trace ='enabled=on'(默认修改当前会话)

​	执行某一个sql之后，使用：select * from information_schema.optimizer_trace来查看优化过程

**更新语句的执行过程**

![image-20220715155130445](C:\Users\tracy\AppData\Roaming\Typora\typora-user-images\image-20220715155130445.png)

	1. 客户端发起修改指令
	1. server从内存buffer pool或者硬盘里读取数据
	1. InnoDB存储引擎执行器修改值，并更新内存
	1. InnoDB存储引擎记录redolog防止mysql的服务崩溃，并将该行记录的状态设置未prepare，等待真正写入成功
	1. InnoDB存储引擎告诉server可以提交事务
	1. server将修改操作记录到binlog
	1. InnoDB操作redolog，修改记录状态为commit

**文件存储结构（宏观）**``这里注意:8.0取消了frm的文件``

<img src="C:\Users\tracy\AppData\Roaming\Typora\typora-user-images\image-20220714101855825.png" alt="image-20220714101855825" style="zoom:67%;" />

**用户管理**

​	//  8.0之后的版本都是用cache_sha2_password插件加密，可能会引起升级后不兼容的问题

​	create user 'username' identified  ``with mysql_native_password`` by 'password'; 

​	// 修改用户

​	alter user 'username' identified by 'new_password';

​	// 解锁用户

​	alter user 'username' account unlock;

​	flush privilges;

​	// 设置密码过期、失败尝试次数等

​	create user ‘username’ xxxx

​	生产环境中，慎用删除用户操作，因为你不清楚用户的权限，但是你可以先锁定该用户。

**忘记密码**

​	// 先关闭mysql服务

​	mysqld stop；            有的系统是：service mysqld stop

​	// 跳过授权启动

​	mysqld_safe --skip-grant-tables --skip-networking &

​	// 加载授权表

​	flush privilges;

​	// 修改用户密码

​	alter user root identified by 123;

​	//重新启动mysql

​	mysqld restart;           有的系统是： service mysqld restart;

**日志管理**

​	**error log:** 从mysql启动到生命周期结束记录错误

​		show variables like '%log_error%';

​	**general log**: 普通日志，记录mysql的所有操作语句，用于诊断和调试

​		

​	**binlog:** 记录mysql修改类的sql日志,DDL（8.0默认开启，5.6、5.7需要手动开启）

​		可以做数据恢复、主从、审计（需要工具解析日志、binlog2sql、my2sql）

​		配置binlog: 在my.cnf文件中配置log_bin=path，重启mysql

​	**slow log**: 默认不开启

​		slow_query_log=1,slow_query_log_file=xxxx

​		维度信息：

​				long_query_time=0.5(s),

​				long_queries_not_using_indexs=1,

​				long_throttle_queries_not_using_indexs

​		

