# mysql事务实现原理

## 1. summary

- 事务的隔离性

  锁 and MVCC

- 事务的原子性和持久性

  Redo Log

- 事务的一致性

  Undo log

## 2. Redo log

- innodb engine中产生

- 主要记录物理日志，即对磁盘上的数据进行的修改操作

- 用于恢复提交后的物理数据页， but只能恢复到最后一次提交的位置

  - 已提交但未写入ibd文件中的数据

- 分为

  - redo log buffer，内存中，易丢失
  - redo log file， 磁盘中，持久化

  先写入redo log buffer，再根据一定的策略刷到rodo log file中实现持久化

  - 事务提交后每秒将buffer的数据写入内核缓冲并且fsync刷入磁盘。(0)
  - 事务提交时将buffer数据写入内核缓冲区并且调用fsync将日志写到磁盘上，这样可以保证数据不丢失。(1)
  - 事务提交时将buffer写入内核缓冲区，之后每条调用fsync将内核缓存刷入磁盘。(2)

  ![1](./images/事务-1.png)

  

- 刷盘机制

  - 开启事务时，根据`innodb_flush_log_at_trx_commit`决定，即上图中的0，1，2
  - 每秒刷新一次，频率根据`innodb_flush_log_at_timeout`决定，default=1s
  - 当log buffer已经使用的内存超过一半
  - writePos追上了checkPoint

- 写入机制

  循环覆盖写

  - writePos

    当前记录所在位置

  - checkPoint

    当前擦除所在位置（已持久化到磁盘）

  - writePos和checkPoint之间

    可写入数据的位置

- LSN机制

  除了存在于redo log外，还存在于数据页

  - 每个数据页的头部，有参数记录了当前页最终的LSN
  - 将数据页的LSN和redo log中的LSN进行对比
    - 若数据页LSN < redo log LSN, 则表示丢失了部分数据
    - 需要用redo log来恢复数据

## 3. Undo log

- 作用

  - 回滚事务

    启动事务前，会先将要修改的数据记录存储到undo log，若需要回滚，根据undo log对未提交的事务进行回滚

    事务提交后，将当前事务对应的undo log放入待删除列表，后续由后台进程进行删除

  - 实现MVCC

    数据被别的事务锁定时，可以根据undo log分析出当前数据之前的版本，返回给客户端

- 记录的是逻辑日志（与操作相反的语句）

- 事务执行过程中产生的undo log也需要进行持久化

  - 产生redo log
  - 因此，数据库崩溃时先做redo log数据恢复，再做undo log 回滚

- 存储方式

  采用段进行管理

- 如果实现MVCC

  - undo log 版本链

    undo log中的每一条数据都会有记录指向上个版本的undo log

    当一行数据有多个版本时，就会有多条 undo log 日志，undo log 之间通过 roll_pointer 指针连接，这样就形成了一个 undo log 版本链

  - 根据read view和undo log 版本链实现MVCC

    https://www.modb.pro/db/78749

## 4. binlog

- mysql自身的日志

- 记录所有mysql数据库表结构/数据变更的二进制日志

  不记录select/show这类查询操作日志

  - 以事件形式记录相关变更

- 作用

  - 主从复制
  - 数据恢复

- 记录模式

  - row

    记录每一行数据被修改的情况

  - statement

    只记录sql语句

    有时会出现主从不一致

  - mixed

    一般使用statement，若出现可能不一致的语句，则使用row

- 写入机制

  1. 根据记录模式和操作触发事件生成日志事件

  2. 将事务执行过程中产生的日志事件写入相应的缓冲区

     每个事物线程一个缓冲区

  3. 事务在commit阶段会将产生的日志事件写入磁盘的binlog文件中

     不用事务以船行的方式将日志事件写入binlog中

     so一个事务中包含的日志事件信息在binlog中是连续的

## 5. binlog V.S. redo log

| binlog                                               | redo log                                                     |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| mysql本身拥有的，不管使用何种engine，binlog都在      | innodb特有的                                                 |
| 逻辑日志，记录对数据库的所有修改操作                 | 物理日志，记录对每个数据页的修改                             |
| 不具有幂等，记录的是所有影响数据库的操作             | 具有幂等                                                     |
| 开启事务时，将每次提交的事务一次性的写入内存缓冲区   | 在数据准备修改之前将数据写入缓冲区的redo log中，然后在缓冲区修改数据。提交事务时，先将redo log写入缓冲区，写入完成后提交事务 |
| 一个事务的binlog中间不会插入其他事务的binlog         | 一个事务的redo log中间会插入其他事务的redo log               |
| 追加写入，不会覆盖使用                               | 循环使用，日志大小固定                                       |
| 一般用于主从复制和数据恢复，不具备崩溃自动恢复的能力 | 服务器发生故障后重启mysql，用于恢复事务已提交但未写入数据表的数据 |


