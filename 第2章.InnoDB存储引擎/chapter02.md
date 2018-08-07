#InnoDB体系架构
##后台线程
###Master Thread
负责缓冲池中的数据异步刷新到磁盘，保证数据一致性，包括脏页的刷新、合并插入缓冲、UNDO页的回收等。
###IO Thread
write、read、insert buffer、log thread四种。
###Purge Thread
从Master线程中独立出来，用于合并已分配且使用的undo页。
###Page Clean Thread
从Master线程中独立出来，用于脏页的刷新。
##内存
###缓冲池
缓冲池中存储的数据页类型有：索引页、数据页、undo页、插入缓冲（insert buffer）、自适应哈希（adaptive hash index）、索信息、数据字典信息等。
###LRU、Free、Flush List
####LRU
在传统的LRU算法基础上，增加了midpoint位置，读取到的页，虽然是最新访问的页，但并不是直接放入到LRU的首部，而是放入LRU列表的midpoint位置。默认配置为5/8左右。
####Free
LRU用来管理已经读取的页，数据库刚启动时，LRU列表是空的，这时所有的页都在Free List。

当需要从缓冲池分配页时，首先从Free List查看是否有可用空闲页，若有则从Free List删除，然后加入到LRU List；否则，淘汰LRU末尾的页，并将该空间分配给新的页。
####Flush
在LRU List中的页被修改以后，就变成了脏页(dirty page)，即缓冲池中的页和磁盘上的页数据产生了不一致。这时数据库通过Checkpoint机制把脏页刷回磁盘，而Flush List中的页即为脏页列表。

脏页既存在于LRU List也存在与Flush List。

LRU List用来管理缓冲池中页的可用性，Flush List用来管理将页刷回磁盘，保证一致性，二者互不影响。
###重做日志缓冲
InnoDB引擎首先把重做日志信息放入到这个缓冲区，然后按照一定频率将其刷新到重做日志文件。

重做日志缓冲不需要设置的很大，因为一般情况下每1秒钟都会将重做日志缓冲刷新到日志文件。

重做日志缓冲在三种情况下回刷新到重做日志文件：

1、Master Thread每一秒都将重做日志缓冲刷新到重做日志文件。

2、每个事务提交时会将...。

3、当重做日志缓冲剩余空间小于1/2时，...。
###额外内存池
#Checkpoint技术
缓冲池的目的是协调CPU速度与磁盘速度的鸿沟。因此，页的操作首先在缓冲池完成，一条DML语句改变了页中的记录，那么此时页是脏的，即：缓冲池中的页的版本比磁盘的要新。数据库需要将新版本的页从缓冲池刷新到磁盘。

WAL，Write Ahead Log策略，在事务提交时，先写事务日志，再修改页。满足事务的持久性（Durability）。

Checkpoint技术解决的问题：

##缩短数据库的恢复时间；
数据库发生宕机恢复时，不需要重做所有的日志，因为checkpoint点之前的页都已经刷回磁盘。只需要对checkpoint点之后的重做日志进行恢复。
##缓冲池不够用时，将脏页刷到磁盘；
缓冲池不够用时，根据LRU算法溢出最近最少使用的页，如果此页为脏页，则强制进行checkpoint，把脏页刷回磁盘。
##重做日志不可用时，刷新脏页；
重做日志是两个可以重用的文件，大小并不是无限增大。
##LSN
InnoDB引擎通过LSN（log sequence number）来标识版本，每个lsn有8个字节，共64位。

页有lsn、redo log有lsn、checkpoint有lsn。

#InnoDB关键特性

