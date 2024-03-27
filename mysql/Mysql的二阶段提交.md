---
title: Mysql的二阶段提交
date: 2023-08-25 17:40:00
tags: mysql
categories: mysql
---

## Mysql的二阶段提交

Mysql在事务提交时，会往`bin log`和`redo log`写日志。我们知道`bin log`是主要是用来做主从节点之间的数据同步，而`redo log`主要做崩溃后的数据恢复。如果两种日志少了一个，那么都会导致数据不一致。

那么Mysql会先写哪个日志呢？

+ 假设Mysql先写`bin log`：如果在写完`bin log`之后，写完`redo log`之前Mysql崩溃。Mysql依靠`redo log`做数据恢复，发现事务无效。但`bin log`已经记录，从库通过主库`bin log`做数据同步，就出现了数据不一致。
+ 假设Mysql先写`redo log`：如果在写完`redo log`之后，写完`bin log`之前Mysql崩溃。Mysql依靠`redo log`做数据恢复，数据的更改生效，但`bin log`中没有记录。从库通过主库`bin log`做数据同步，就出现了数据不一致。

那Mysql是怎么保证数据一致的呢？通过2PC

### 2PC

Mysql把`redo log`写入分为两阶段，准备(prepare)阶段和提交(commit)阶段。整个流程变成了

![2PC(二段提交)](../img/Mysql的二阶段提交.assets/2PC(二段提交).svg)

Mysql先往`redo log`里写入日志，事务状态为`prepare`，然后往`bin log`里写入日志，最后把`redo log`中日志的事务状态改为`commit`。

那2PC能否解决数据不一致的问题呢？

+ `prepare`阶段写`redo log`时崩溃：事务无效，没有`bin log`，数据修改不生效。
+ `prepare`阶段写`redo log`完毕，写`bin log`时崩溃：事务无效，没有`bin log`，数据修改不生效。
+ 写`bin log`完毕，`commit`时崩溃：`redo log`中事务处于`prepare`，会根据事务id去`bin log`查询，判断`bin log`中的事务日志是否完整，如果完整就提交事务。否则回滚。

Mysql通过`bin log`和`redo log `的二阶段提交来保证数据一致性。
