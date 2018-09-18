**2PC是InnoDB对事物提交的机制；**
MySQL开启BinLog以后，与InnoDB之间的同步属于内部XA事务问题；（内部分布式事务）
XA和2PC基本上可以认为一回事！

**XA-2PC (two phase commit, 两阶段提交 )**
第一阶段：为prepare阶段，TM向RM发出prepare指令，RM进行操作，然后返回成功与否的信息给TM；
第二阶段：为事务提交或者回滚阶段，如果TM收到所有RM的成功消息，则TM向RM发出提交指令；不然则发出回滚指令；

MySQL通过两阶段提交很好地解决了binlog和redo log的一致性问题
第一阶段：InnoDB prepare，持有prepare_commit_mutex，并且write/sync redo log； 将回滚段设置为Prepared状态，binlog不作任何操作；
第二阶段：包含两步
1、write/sync Binlog；
2、InnoDB commit (写入COMMIT标记后释放prepare_commit_mutex)；

以 binlog 的写入与否作为事务提交成功与否的标志，innodb commit标志并不是事务成功与否的标志。因为此时的事务崩溃恢复过程如下：
1、崩溃恢复时，扫描最后一个Binlog文件，提取其中的xid； 
2、InnoDB维持了状态为Prepare的事务链表，将这些事务的xid和Binlog中记录的xid做比较，如果在Binlog中存在，则提交，否则回滚事务。
通过这种方式，可以让InnoDB和Binlog中的事务状态保持一致。如果在写入innodb commit标志时崩溃，则恢复时，会重新对commit标志进行写入；
在prepare阶段崩溃，则会回滚，在write/sync binlog阶段崩溃，也会回滚。这种事务提交的实现是MySQL5.6之前的实现。

**MySQL5.6中的binlog 组提交**
将Binlog Group Commit的过程拆分成了三个阶段
flush stage 将各个线程的binlog从cache写到文件中; 
sync stage 对binlog做fsync操作（如果需要的话；最重要的就是这一步，对多个线程的binlog合并写入磁盘）；
commit stage 为各个线程做引擎层的事务commit(这里不用写redo log，在prepare阶段已写)。
每个stage同时只有一个线程在操作。(分成三个阶段，每个阶段的任务分配给一个专门的线程，这是典型的并发优化)
这种实现的优势在于三个阶段可以并发执行，从而提升效率。注意prepare阶段没有变，还是write/sync redo log.

**MySQL5.7中的binlog组提交**
从XA恢复的逻辑我们可以知道，只要保证InnoDB Prepare的redo日志在写Binlog前完成write/sync即可
InnoDB Prepare，记录当前的LSN到thd中；
进入Group Commit的flush stage；Leader搜集队列，同时算出队列中最大的LSN。 
将InnoDB的redo log write/fsync到指定的LSN
写Binlog并进行随后的工作(sync Binlog, InnoDB commit , etc)
也就是将 redo log的write/sync延迟到了binlog group commit的 flush stage之后，sync binlog之前。
通过延迟写redo log的方式，显式的为redo log做了一次组写入(redo log group write)，并减少了(redo log) log_sys->mutex的竞争。