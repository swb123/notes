## InnoDB存储引擎

&emsp;InnoDB是事务安全的MySQL存储引擎，通常来说，InnoDB存储引擎是OLTP应用中核心表的首选存储引擎。

### InnoDB存储引擎概述

&emsp;InnoDB存储引擎是第一个完整支持ACID事务的MySQL存储引擎，其特点是行锁设计、支持MVCC、支持外键、提供一致性非锁定读，同时被设计用来最有效地利用以及使用内存和CPU。

### InnoDB体系架构

![1569742982845](C:\Users\施文波\AppData\Roaming\Typora\typora-user-images\1569742982845.png)

&emsp;由图可知，InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

+ 维护所有进程/线程需要访问地多个内部数据结构。
+ 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
+ 重做日志（redo log）缓存。

&emsp;后台线程的主要作用是负责刷新内存池中的数据保证缓存池中的内存缓存的是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行的状态。

#### 后台线程

&emsp;InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。

1. Master Thread

   Master Thread 是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页的回收。

2. IO Thead

   在InnoDB存储引擎中大量使用了AIO(Async IO)来处理写IO请求，极大地提高了数据库行呢个。而IO Thread的工作主要是负责这些IO请求的回调（call back）处理，分为write、read、insert buufer和log IO thread。

3.  Purge Thread

   事务被提交后，其所使用的undolog可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。

4. Page Cleaner Thread

   Page Cleaber Thread 是在InnoDB1.2x版本中引入的。其作用是将之前版本中的脏页刷新操作都放入到单独的线程中来完成。其目的是为了减轻原Master Thread的工作及对于用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能。

#### 内存

1. **缓冲池**

   &emsp;InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲技术来提高数据库的整体性能。

   &emsp;缓冲池简单来说是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。在数据库中进行读取页的操作，首先将从磁盘读到的页存放到缓冲池中，这个过程称为将页“FIX”在缓冲池中。下一次再读相同的页时，首先判断该页是否再缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页。

   &emsp;对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。页从缓冲池刷新回到磁盘的操作并不是每次页发生更新时操作，而是通过一种称为Checkingpoint的机制刷新回磁盘。

   &emsp;具体来看，缓冲池中缓冲的数据页类型有：索引页、数据页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等。下图显示了InnoDB存储引擎中内存的结构情况。

   ![1569745297439](C:\Users\施文波\AppData\Roaming\Typora\typora-user-images\1569745297439.png)

   &emsp;从InnoDB 1.0x版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同缓冲池实例中。这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力。

2. **LRU List、Free List和Flush List**

   &emsp;数据库中的缓冲池是通过LRU(Latest Recent Used，最近最少使用)算法来进行管理。即最频繁使用的页再LRU列表的前端，而最少使用的页在LRU列表的尾端，当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页。

   &emsp;在InnoDB存储引擎中，缓冲池中页的大小默认为16KB，同样使用LRU算法对缓冲池进行管理。在InnoDB的存储引擎中，LRU列表中还加入了midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LRU列表的首部，而是直接放入到 LRU列表的midpoint位置。这个算法在InnoDB存储引擎下称为midpoint insertion strategy。在默认配置下，该位置在LRU列表长度的5/8处。

   &emsp;在InnoDB存储引擎中，把midpoint之后的列表称为old列表，之前的列表称为new列表。

   &emsp;为什么不采用朴素的LRU算法，直接将读取的页放入到列表的首部呢？这是因为若直接将读取到的页放入到LRU的首部，那么某些SQL操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。常见的这类操作为索引或数据的扫描操作。这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说又仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU列表的首部，那么非常可能将所需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，InnoDB存储引擎需要再次访问磁盘。

   &emsp;InnoDB存储引擎引入了另一个参数来进一步管理LRU列表，这个参数是innodb_old_blocks_time，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。

   &emsp;LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何的页。这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入到LRU列表中。否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页。当页从LRU列表的old部分加入到new部分时，称此时发生的操作为page made young，而因为innodb_old_blocks_time的设置而导致页没有从old部分移动到new部分的操作称为page not made young。

   &emsp;InnodeDB存储引擎从1.0x版本开始支持压缩页的功能，即将原本16KB的页压缩为1KB、2KB、4KB和8KB。而由于页的大小发生了变化，LRU列表也有了些许的改变。对于非16KB的页，是通过unzip_LRU列表进行管理的。

   &emsp;对于压缩页的表，每个表的压缩比率可能个不相同。可能存在有的表页大小为8KB，有的表页大小为2KB的情况。unzip_LRU是怎么从缓冲池中分配内存呢？

   &emsp;首先，在unzip_LRU列表中对不同压缩页大小的页进行分别管理。其次，通过伙伴算法进行内存的分配。例如对需要从缓冲池中申请页为4KB的大小，其过程如下：

   1).  检查4KB的unzip_LRU列表，检查是否有可用的空闲页；

   2).  若有，则直接使用；

   3). 否则，检查8KB的unzip_LRU列表；

   4).  若能够得到空闲页，将页分成2个4KB页，存放4KB的unzip_LRU列表；

   5).  若不能得到空闲页，从LRU列表中申请一个16KB的页，将页分成1个8KB的页、2个4KB的页、分别存放到对应的unzip_LRU列表中。

   &emsp;在LRUl列表中的页被修改后，称该页为脏页，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过CHECKPOINT机制将脏页刷新回磁盘。而Flush列表中的页即为脏页列表。脏页既存在于LRU列表中，也存在于Flush列表中。LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷新回磁盘，二者互补影响。

3. **重做日志缓冲**

   &emsp;InnoDB存储引擎的内存区域除了有缓冲池，还有重做日志缓冲（redo log buffer）。InnoDB存储引擎首先将重做日志信息放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志缓冲一般不需要设置很大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒产生的事务量在这个缓冲大小之内即可。

   &emsp;重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中。

   + Master Thread每一秒将重做日志缓冲刷新到重做日志文件；
   + 每个事务提交时会将重做日志缓冲刷新到重做日志文件；
   + 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件。

4. 额外的内存池

   在InnoDB存储引擎中，对内存的管理是通过一种称为内存堆（heap）的方式进行的。在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域的内存不够时，会从缓冲池进行申请。

   ### Checkpoint技术

   &emsp;为了避免发生数据丢失的问题，当前事务数据库系统普遍都采用了Write Ahead Log策略，即当事务提交时，先写重做日志，再修改页。当由于发生宕机而导致数据丢失时，通过重做日志来完成数据的恢复。这也是事务ACID中D(Durability 持久性)的要求。

   &emsp;Checkpoint(检查点)技术的目的是解决以下几个问题：

   + 缩短数据库的恢复时间；
   + 缓冲池不够用时，将脏页刷新到磁盘；
   + 重做日志不可用时，刷新脏页

   &emsp;当数据库发生宕机时，数据库不需要重做所有的日志，因为CheckPoint之前的页都已经刷新回磁盘。故数据库只需对Checkpoint后的重做日志进行恢复。这样就大大缩短了恢复时间。

   &emsp;当缓冲池不够用时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘。

   &emsp;重做日志出现不可用的情况是因为当前事务数据库系统对重做日志的设计都是循环使用的，并不是让其无限增大的，重做日志可以被重用的部分是指这些重做日志已经不再需要，即当数据库发生宕机时，数据库恢复操作不需要这部分的重做日志，因此这部分就可以被覆盖重用。若此时重做日志还需要使用，那么必须强制产生Checkpoint，将缓冲池中页至少刷新到当前重做日志的位置。

   &emsp;再InnoDB存储引擎内部，有两种Checkpoint，分别为：

   + Sharp Checkpoint
   + Fuzzy Checkpoint

   &emsp;Sharp Checkpoint 发生在数据库关闭时将所有的脏页都刷新回磁盘，这时默认的工作方式，即参数innodb_fast_shutdown = 1;

   &emsp;但是，若是若数据库在运行时也使用Sharp Checkpoint，那么数据库的可用性就会受到很大的影响。故在InnoDB存储引擎内部使用Fuzzy Checkpoint进行页的刷新，即只刷新一部分脏页，而不是刷新所有的脏页回磁盘。

   &emsp;在InnoDB存储引擎中可能发生如下几种情况的Fuzzy Checkpoint:

   + Master Thread Checkpoint
   + FLUSH_LRU_LIST Checkpoint
   + Async/Sync FLush Checkpoint
   + Diry Page too much Checkpoint

   &emsp;Master Thread中发生Checkpoint，差不多以每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘。这个过程是异步的，即此时InnoDB存储引擎可以进行其他的操作，用户查询线程不会阻塞。

   &emsp;FLUSH_LRU_LIST Checkpoint是因为InnoDB存储引擎需要保证LRU列表中需要有差不多100个空闲页可供使用。倘若没有100个可用空闲页，那么InnoDB存储引擎会将LRU列表尾端的页移除。如果这些页中有脏页，那么需要进行Checkpoint，这些页是来自LRU列表的，因此称为FLUSH_LRU_LIST Checkpoint。而从MySQL5.6版本开始，这个检查被放在了一个单独的Page Cleaner线程中进行，并且用户可以通过参数innodb_lru_scan_depth控制LRU列表中可用也页的数量，该值默认为1024。

   &emsp;Async/Sync Flush Checkpoint指的是重做日志文件不可用的情况，这时需要强制将一些页刷新会磁盘，而此时脏页是从脏页列表中选取的。

   &emsp;最后一种Checkpoint的情况是Dirty Page too much，即脏页的数量太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总的来说还是为了保证缓冲池中有足够的可用的页。

   ### Master Thread 工作方式

   #### InnoDB 1.0x版本之前的Master Thread

   &emsp;Master Thread 具有最高的线程优先级别。其内部由多个循环组成：主循环（loop）、后台魂环(backgroup loop)、刷新循环(flush loop)、暂停循环(suspend loop)。Master Thread 会根据数据库运行的状态在loop、backgroud loop、flush loop和suspend loop中进行切换。

   1. Loop被称为主循环，其中有两大部分的操作----每秒钟的操作和每10秒的操作。

      每秒一次的操作包括：

      + 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）；
      + 合并插入缓冲(可能)；
      + 至多刷新100个InnoDB的缓冲池中的脏页到磁盘(可能)；
      + 如果当前没有多个用户活动，则切换到background loop(可能)。

      每10秒的操作，包括如下内容：

      + 刷新100个脏页到磁盘（可能情况下）；
      + 合并至多5个插入缓冲(总是)；
      + 将日志缓冲刷新到磁盘(总是)；
      + 删除无用的Undo页(总是)；
      + 刷新100个或者10个脏页到磁盘(总是)。

   2. backgroup loop，若当前没有用户活动（数据库空闲时）或者数据库关闭(shutdown)，就会切换到这个循环。background loop会执行以下操作：

      + 删除无用的Undo页（总是）；
      + 合并20个插入缓冲(总是)；
      + 跳回到主循环(总是)；
      + 不断刷新100个页直到符合条件(可能，跳转到flush loop中完成)。

   #### InnoDB1.2版本之前的Master Thread

    &emsp;innodb_io_capacity表示磁盘IO的吞吐量，默认值为200。

   + 在合并插入缓冲时，合并插入缓冲的数量为innodb_io_capacity值的5%；
   + 在从缓冲区刷新页时，刷新脏页的数量为innodb_io_capacity。

### InnoDB的关键特性

InnoDB存储引擎的关键特性包括：

+ 插入缓冲(Insert Buffer)
+ 两次写(Double Write)
+ 自适应哈希索引(Adaptive Hash Index)
+ 异步IO(Async IO)
+ 刷新邻接页(Flush Neighbor Page)

#### 插入缓冲

1. **Insert Buffer**

   &emsp;在InnoDB存储引擎中，主键是唯一的标识符。通常应用程序中行记录的插入顺序是按照主键递增顺序进行插入的。因此，插入聚集索引(Primary Key) 一般是顺序的，不需要磁盘的随机读取。

   &emsp;但是不可能每张表上只有一个聚集索引，更多情况下，一张表上有多个非聚集的辅助索引。在进行插入操作时，数据页的存放还是按照主键进行顺序存放，但是对于非聚集索引叶子节点的插入不再时顺序的了，这时就需要离散地访问非聚集索引页，由于随机读取的存在导致了插入操作性能下降。这是因为B+树的特性决定了非聚集索引插入的离散性。

   &emsp;InnoDB存储引擎开创性地设计了Insert Buffer，对于非聚集索引地插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer对象中。数据库这个非聚集的索引已经插到叶子节点，而实际并没有，只是存放在另一个位置。然后再以一定的频率和情况进行Insert Buffer和辅助索引页叶子节点的merge操作，这时通常能将多个插入合并到一个操作中(因为在一个索引页中)，这就大大提高了对于非聚集索引插入的性能。

   &emsp;然而Insert Buffer的使用需要同时满足以下两个条件：

   + 索引是辅助索引；
   + 索引不是唯一的。

   &emsp;目前Insert Buffer 存在一个问题是：在写密集的情况下：插入缓冲会占用过多的缓冲池内存，默认最大可以占用到1/2的缓冲池内存。

2. **Change Buffer**

   InnoDB从1.0x版本开始引入了Change Buffer，可以将其视为Insert Buffer的升级。从这个版本开始，InnoDB存储引擎可以对DML操作——INSERT、DELETE、UPDATE都进行缓冲，他们分别是：Insert Buffer、Delete Buffer、Purge buffer。

   &emsp;Change Buffer适用的对象依然是非唯一的辅助索引。

   &emsp;对一条记录进行UPDATE操作可能分为两个过程：

   + 将记录标记为已删除；

   + 真正将记录删除。

     因此Delete Buffer对应UPDATE操作的第一个过程，即将记录标记为删除。Purge Buffer对应UPDATE操作的第二个过程，即将记录真正的删除。

3. **Insert Buffer的内部实现**

   &emsp;Insert Buffer的数据结构是一颗B+树。全局只有一颗Insert Buffer B+树，负责对所有表的辅助索引进行Insert Buffer。而这棵B+树存放在共享表空间，默认也就是ibdatal中。

   &emsp;Insert Buffer是一棵B+树，因此从其也由叶节点和非叶节点组成。非页节点存放的是查询的search key(键值)，其构造如图所示：

   ![1569944429802](C:\Users\施文波\AppData\Roaming\Typora\typora-user-images\1569944429802.png)

   &emsp;search key一共占用9个字节，其中space表示待插入记录所在表空间的id，在InnoDB存储引擎中，每个表有一个唯一的space id，可以通过space id查询得知是哪张表。space占用4字节，marker占用1字节，它用来兼容老版本的Insert Buffer。offset 表示页所在的偏移量，占用4字节。

   &emsp;当一个辅助索引要插入到页（space, offset）时，如果这个页不在缓冲池中，那么InnoDB存储引擎首先根据上述规则构造一个search key，接下来查询Insert Buffer这棵B+树，然后再将这条记录插入到Insert Buffer B+树的叶子节点中。

   &emsp;对于插入到Insert Buffer B+树叶子节点的记录，并不是直接将待插入的记录插入，而是需要根据如下的规则进行构造：

   ![1569944965844](C:\Users\施文波\AppData\Roaming\Typora\typora-user-images\1569944965844.png)

   &emsp;space、marker、page_no字段和之前非叶节点中的含义相同，一共占用9字节。第4个字段metadata占用4字节，其存储内容如下表所示：

   ![1569945198069](C:\Users\施文波\AppData\Roaming\Typora\typora-user-images\1569945198069.png)

4. **Merge Insert Buffer**

   &emsp;Merge Insert Buffer的操作可能发生在以下几种情况下：

   + 辅助索引页被读取到缓冲池时；
   + Insert Buffer Bitmap页追踪到该辅助索引页已无可用空间时：
   + Master Thread

#### 两次写

&emsp;为了避免部分写失败导致数据丢失，出席拿了两次写--doublewrite。在应用重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做，这就是doublewrite。

&emsp;doublewrite 由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续的128个页，即2个区(extent)，大小同样为2MB。在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是会通过memcpy函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer 再分两次，每次1MB顺序地写入共享表空间地物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。在这个过程中，因为doublewrite页时连续的，因为这个过程是顺序写的

































