>原子性
>
>一致性
>
>隔离性
>
>持久性
#7.1 认识事务
虽然事务的定义极其严格：ACID，但是不同的数据库厂商出于各种目的，并没有去严格的遵守ACID。

#7.2 事务的实现
MySQL中对于ACID的实现：

隔离性：通过锁实现。

原子性、持久性：redo。

一致性：undo log。

> Redo Log：恢复某个已提交事务修改的页操作；物理日志：记录页的修改情况
>
>Undo Log：回滚某一行数据到某个特定版本。逻辑日志：保存行的旧版本
##Redo Log
redo log基本上都是顺序写，数据库运行时不需要对redo log进行读取操作。而undo log则会进行随机读写。

innodb_flush_log_at_trx_commit：</br>
1：事务提交时，把重做日志缓冲写入到重做日志文件，并调用一次fsync操作；</br>
0：提交时不写重做日志文件，由master thread定期去做。</br>
2：事务提交时，把重做日志写入重做日志文件，但不执行fsync。</br>

设置为0和2会提高性能，但是会丧失持久性。

|XXX|Binlog|Redolog|
|---|---|---|
|写入时机|事务执行过程中，binlog event不断写入binlog_cache，</br>事务提交时，把cache写入文件|事务执行过程中不断写入redolog buffer，提交时写入到文件|
![写入时机](./png/BinRedoDiff.png)

define LOG_BLOCK_CHECKPOINT_NO 8 /* 4 lower bytes of the value of

###LSN:
重做日志写入的总量：可以理解为系统的逻辑时钟。

checkpoint的位置: 在第一个redo log文件头部进行记录。

页的版本--FIL_PAGE_LSN，page_header有以上结构，更改页的记录时的lsn。
![LSN](./png/LSN.gif)
创建阶段：事务创建一条日志；

日志刷盘：日志写入到磁盘上的日志文件；

数据刷盘：日志对应的脏页数据写入到磁盘上的数据文件；

写CKP：日志被当作Checkpoint写入日志文件；

对应这4个阶段，系统记录了4个日志相关的信息，用于其它各种处理使用：

Log sequence number（LSN1）：当前系统LSN最大值，新的事务日志LSN将在此基础上生成（LSN1+新日志的大小）；

Log flushed up to（LSN2）：当前已经写入日志文件的LSN；

Oldest modified data log（LSN3）：当前最旧的脏页数据对应的LSN，写Checkpoint的时候直接将此LSN写入到日志文件；

Last checkpoint at（LSN4）：当前已经写入Checkpoint的LSN；

