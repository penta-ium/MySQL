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
#Checkpoint技术
#InnoDB关键特性
gj

