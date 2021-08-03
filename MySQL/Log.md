日志是 MySQL 数据库的重要组成部分，记录着数据库运行期间各种状态信息。MySQL 日志主要包括错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。作为开发，我们重点需要关注的是二进制日志 (`binlog`) 和事务日志 (包括`redo log`和`undo log`)。



# redo log

事务的四大特性里面有一个是**持久性**，具体来说就是**只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态**。

保证持久性，最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有严重的性能问题。

为了解决这个问题，MySQL 使用了 **WAL (Write-Ahead Logging)** 技术。它的关键点就是先记录日志到内存，攒一波再写磁盘。

具体来说，当有一条记录需要更新的时候，InnoDB 引擎会

1. 先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。
2. 同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面。这个更新往往是在系统比较空闲的时候做，就像打烊以后掌柜做的事。



#### 用途

1. 提高持久化时写硬盘的性能
2. 崩溃恢复时恢复最近数据
   1. `redo log file` 记录着 xxx 页做了 xxx 修改，所以即使 `MySQL` 发生宕机，也可以通过 `redo log` 进行数据恢复，也就是说在内存中更新成功后，即使没有刷新到磁盘中，但也不会因为宕机而导致数据丢失。



#### 基本概念

`redo log` 包括两部分：

1. 内存中的日志缓冲 (`redo log buffer`)
2. 磁盘上的日志文件 (`redo log file`)

MySQL 每执行一条 `DML` 语句，先将记录写入 `redo log buffer`，后续某个时间点再一次性将多个操作记录写到 `redo log file`。



在计算机操作系统中，用户空间下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过 内核缓冲区 (`os buffer`)。因此，`redo log buffer` 写入 `redo log file` 实际上是先写入 `os buffer`，然后再通过系统调用 `fsync()` 将其刷到 `redo log file` 中。



MySQL 支持三种将 `redo log buffer` 写入 `redo log file` 的时机，可以通过`innodb_flush_log_at_trx_commit` 参数配置：
- `0` 延迟写
  - 每秒执行一次将 `redo log buffer` 写入 `Os buffer` 及 `fsync()` 的操作
- `1` 实时写，实时刷
  - 实时写入 `os buffer` 并执行 `fsync()`
- `2` 实时写，延时刷
  - 实时写入 `os buffer` ，但每秒才执行一次 `fsync()`



#### 记录形式

InnoDB 的 redo log 是一个固定大小的**循环队列**。比如可以配置为一组 4 个文件，每个文件的大小是 `1GB`，那么总共就可以记录 `4GB` 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![img](assets/v2-9f116e0a858410d0445d797a65d98c4a_720w.jpg)

`write pos` 可视为队列的头节点，是当前记录的位置，一边写一边后移。`checkpoint` 则是队列的尾结点，记录上一次刷盘的位置。

`write pos` 和 `checkpoint` 之间的是空闲空间，可以用来记录新的操作。如果 `write pos` 追上 `checkpoint`，表示队列满了，这时候不能再执行新的更新，得停下来先把数据更新到硬盘，然后把 `checkpoint` 推进一下。

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 **crash-safe**。



#### 记录格式

redo log 是 **物理日志**，记录的是某个 `page` 上某行数据的变化，可以理解为数据行最新的快照。

redo log 以块为单位进行存储，每块占 `512B`，称为 `redo log block`。每个块由3部分组成：日志块头、日志块尾，和日志主体。

```go
type redo log block struct {
    log block header struct {  // 日志块头，12B
        LOG_BLOCK_HDR_NO
        LOG_BLOCK_HDR_LEN
        LOG_BLOCK_FIRST_REC_GROUP
        LOG_BLOCK_CHECKOUTPOINT_NO
    }
    log block                  // 日志主体，492B
    log block tailer struct {  // 日志块尾，8B
        LOG_BLOCK_TRL_NO
    }
}
```



其中 **日志主体** 又分为 4个部分：日志类型、空间ID、页偏移量、和数据部分。

```go
type log block struct {
	type                 // 操作类型（插入/删除）
	space                // 空间ID
	page_no              // 页偏移量，表示第几页
	struct {             // 数据部分
		offset           // 第几行
		len & extra_info // 以下结构只有插入日志才有，删除日志没有
		...
		rec body         // 新的数据
	}
}
```





# binlog

`MySQL` 整体来看，其实有两块：一块是 `Server` 层，它主要做的是功能层面的事情；还有一块是引擎层，负责存储相关的具体事宜。

上面我们聊到的 `redo log` 是 `InnoDB` 引擎特有的日志，而 `Server` 层也有自己的日志，称为 `binlog`（归档日志）。

为什么会有两份日志呢？因为最开始 `MySQL` 里并没有 `InnoDB` 引擎。`MySQL` 自带的引擎是 `MyISAM`，但 `MyISAM` 没有 `crash-safe` 的能力，`binlog` 日志只能用于归档。而 `InnoDB` 是另一个公司以插件形式引入 `MySQL` 的，既然只依靠 `binlog` 是没有 `crash-safe` 能力的，所以 `InnoDB` 使用另外一套日志系统——也就是 `redo log` 来实现 `crash-safe` 能力。

这两种日志有以下三点不同：

1. 层次不同。`redo log` 是 `InnoDB` 引擎特有的；`binlog` 是 `MySQL` 的 `Server` 层实现的，所有引擎都可以使用。
2. 格式不同。`redo log` 是**物理日志**，记录的是 **数据页变更**；`binlog` 是**逻辑日志**，记录的是这个语句的原始逻辑，比如 "给 id=2 这一行的 c 字段加 1 "。
3. 占用空间不同。`redo log` 是循环写的，空间固定会用完；binlog 是可以追加写入的，写到一定大小后会切换到下一个，并不会覆盖以前的日志。
4. 作用不同。 `binlog` 记录的是逻辑，因此可以用于主从同步（从库初始化、从库同步）；`redo log` 记的是最近的数据，可以用于宕机恢复。
5. 写入时机不同，见下



#### UPDATE 执行流程

有了对这两个日志的概念性理解，我们再来看执行器和 InnoDB 引擎在执行这个简单的 `UPDATE` 语句时的内部流程。

1. 执行器先找引擎取 id=2 这一行。id 是主键，引擎直接用树搜索找到这一行。如果这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 `PREPARE` 状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（`COMMIT`）状态，更新完成。



#### 两阶段提交

最后三步看上去有点绕，将 redo log 的写入拆成了两个步骤：`PREPARE` 和 `COMMIT`，这就是**两阶段提交**。

为什么必须有两阶段提交呢？这是为了让两份日志之间的逻辑一致。如果不采用该逻辑，在第二个日志还没有写完期间发生了 crash，会导致两个日志数据不一致，进而导致数据库的状态就 和 用日志恢复出来的库的状态 不一致。

>1. **先写 redo log 后写 binlog。**假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。
>2. **先写 binlog 后写 redo log。**如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。



#### 使用场景

1. 主从复制
2. 数据恢复



#### 刷盘时机

`MySQL` 通过 `sync_binlog` 参数控制 binlog 的刷盘时机，取值范围是 `0-N`：

- 0：不强制要求，由系统自行判断何时写入磁盘
- 1：每次 `COMMIT` 时都要将 binlog 写入磁盘（默认）
- N：每 N 个事务，才会将 binlog写入磁盘

取值越低，一致性越好、性能越差。`MySQL 5.7.7` 之后版本的默认值为 `1`。



#### 日志格式
`binlog` 日志有三种格式，分别为 `STATMENT`、`ROW` 和 `MIXED`。默认为 `ROW`。

- `STATMENT` 
  - 基于 `SQL` 语句的复制  (`statement-based replication, SBR`)，每一条会修改数据的 `SQL` 语句都会记录到 binlog 中 
  - 优点：不需要记录每一行的变化，减少了 binlog 日志量，节约了 `IO`, 从而提高了性能
  - 缺点：在某些情况下会导致主从数据不一致，比如执行获取当前时间命令等

- `ROW` 
  - 基于行的复制 (`row-based replication, RBR`)，不记录每条 `SQL` 语句的上下文信息，仅需记录哪条数据被修改了。
  - 优点：不会出现某些特定情况下的存储过程 / function / trigger 的调用和触发无法被正确复制的问题
  - 缺点：会产生大量的日志，尤其是 `ALTEWR TABLE` 的时候会让日志暴涨

- `MIXED` 
  - 基于`STATMENT`和`ROW`两种模式的混合复制 (`mixed-based replication, MBR`)，一般的复制使用 `STATEMENT`模式，对 `STATEMENT` 模式无法复制的操作使用 `ROW` 模式保存






# undo log

`undo log` 有两个作用：回滚和 `MVCC`。

`undo log` 和 `redo log` 记录物理日志不一样，它是逻辑日志。可以认为当 `delete` 一条记录时，`undo log` 中会记录一条对应的 `insert` 记录，反之亦然，当 `update`  一条记录时，它记录一条对应相反的 `update` 记录。

当执行回滚时，就可以从 `undo log` 中的逻辑记录读取到相应的内容并进行回滚。

应用到行版本控制的时候，也是通过 `undo log` 来实现的：当读取的某一行被其他事务锁定时，它可以从 `undo log` 中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。

`undo log` 默认存放在共享表空间中。可以通过配置，改到别的数据文件里。



#### delete/update操作的内部机制

当事务提交的时候，`innodb` 不会立即删除 `undo log`，因为后续还可能会用到 `undo log`，如隔离级别为 `repeatable read` 时，事务读取的都是开启事务时的最新提交行版本，只要该事务不结束，该行版本就不能删除，即 `undo log` 不能删除。

但是在事务提交的时候，会将该事务对应的 `undo log` 放入到删除列表中，未来通过 `purge` 线程来删除。

`delete` 操作实际上不会直接删除，而是将对象打上 `delete flag`，标记为删除，最终的删除操作是 `purge` 线程完成的。

`update` 分为两种情况：

- 如果更新的不是主键列，在 `undo log` 中直接反向记录是如何 `update` 的。即 `update` 是直接进行的。
- 如果更新的是主键列，则先删除该行，再插入一行目标行，和 `postgreSQL` 一样。





#### 问题

##### 为什么有了 binlog 还需要 redo log ?

1. 历史原因：`innodb` 并不是 `MySQL` 的原生存储引擎。`MySQL` 的原生引擎是 `myISAM`，设计之初就没有支持崩溃恢复。
2. 实现原因：由于 `binlog` 记的是逻辑日志，发生了崩溃之后，是无法凭借 `binlog` 把那些已经提交过的事务进行恢复的（你得找 `binlog offset` 之后执行的数据行，然后从头找这些行的所有记录才能恢复，代价太大）



##### 为什么有了redo log 还需要 binlog ?

`redo log` 是循环写，写到末尾是要回到开头继续写的，也就不包含完整数据，不能用于从库初始化



##### 和 redis 的 rdb / aof 日志比较 ?

`binlog` 的机制和 `aof` 类似，而 `redo log` 更像是 `rdb` 和 `aof` 的结合，它的物理写入像 `rdb`，但持续写入的行为像 `aof`，另外它只存最近的数据，不会存全量



#### 参考

> [林晓斌 - 日志系统：一条SQL更新语句是如何执行的](https://zhuanlan.zhihu.com/p/384074147)
>
> [陈添明 - 必须了解的mysql三大日志](https://juejin.cn/post/6860252224930070536)