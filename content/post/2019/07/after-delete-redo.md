---
title: "删除redo后的恢复"
date: 2019-07-15
draft: false
tags: ["Oracle"]
---

记录一下删除`redo log`后的恢复步骤
<!--more-->


# 恢复分为两种：
- 损坏的redo log是`INACTIVE`
- 损坏的redo log是`ACTIVE`或`CURRENT`


## 损坏的redo log是`INACTIVE`

如果损坏的redo log是`INACTIVE`状态的，也就是实例崩溃恢复用不到的redo log，那处理起来比较容易，直接
```
alter database clear logfile group xx;
或者
alter database clear unarchived logfile group xx;
```


```
SQL> select * from v$log;----可以再次确定redo文件不是当前日志，且已经归档

SQL>alter database clear logfile group xx;
SQL>alter database open;

但如果redo0x还没归档的话，那就需要加unarchived了，
SQL>alter database clear unarchived logfile group xx;
```
重建日志组就行了。
建议重建日志文件级后对数据库做一个全库备份，特别是强制clear后，会造成归档日志文件的断层。


## 损坏的redo log是`ACTIVE`或`CURRENT`

如果损坏的redo log是`ACTIVE`或`CURRENT`状态的，也就是实例崩溃恢复需要用到的redo log，那处理起来就比较麻烦了，损坏这种redo log就意味着丢失数据。

redo log的三种状态：

- INACTIVE：日志对应的修改已经被写入硬盘
- ACTIVE：日志对应的修改还没有被写入硬盘
- CURRENT：实例正在使用的日志文件

当然，如果有备份，那么动作基本上就是  `restore，recover until cancel`的。

下面主要说说强制拉起来的步骤：


### 尝试启动数据库
```
SQL >startup
ORACLE instance started.

Total System Global Area 1603411968 bytes
Fixed Size        2253664 bytes
Variable Size      1476398240 bytes
Database Buffers    117440512 bytes
Redo Buffers          7319552 bytes
Database mounted.
ORA-00313: open failed for members of log group 2 of thread 1
ORA-00312: online log 2 thread 1: '/u01/app/oracle/oradata/orcl/redo02.log'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
```

### 尝试使用clear方式重建日志组出现报错
```
idle>alter database clear logfile group 2;
alter database clear logfile group 2
*
ERROR at line 1:
ORA-01624: log 2 needed for crash recovery of instance orcl (thread 1)
ORA-00312: online log 2 thread 1: '/u02/app/oracle/oradata/orcl/redo02.log'


idle>alter database clear unarchived logfile group 2;
alter database clear unarchived logfile group 2
*
ERROR at line 1:
ORA-01624: log 2 needed for crash recovery of instance orcl (thread 1)
ORA-00312: online log 2 thread 1: '/u02/app/oracle/oradata/orcl/redo02.log'
```

从报错信息中可以看出log 2是实例崩溃恢复所需要的日志文件，不能直接重建。

### 修改隐藏文件并尝试强制打开数据库
这种情况下使用隐含参数`_allow_resetlogs_corruption`，创建pfile，把`*._allow_resetlogs_corruption=TRUE`加入到pfile中。然后mount数据库，强制不完全恢复，再`open resetlogs`

```
idle>create pfile='/home/oracle/initorcl.ora' from spfile;

File created.
$ vi /home/oracle/initorcl.ora
SQL> shutdown immediate;
ORA-01109: database not open


Database dismounted.
ORACLE instance shut down.
SQL> startup pfile='/home/oracle/initorcl.ora' mount;
ORACLE instance started.

Total System Global Area 1603411968 bytes
Fixed Size        2253664 bytes
Variable Size      1476398240 bytes
Database Buffers    117440512 bytes
Redo Buffers          7319552 bytes
Database mounted.
SQL> show parameter _allow_

NAME                    TYPE                VALUE
------------------------------------ --------------------------------- ------------------------------
_allow_resetlogs_corruption      boolean                  TRUE

SQL> alter database open resetlogs;
--这里一定要用resetlogs呀，因为要是不用的话，就是代表要用redo进行实例恢复了，那肯定会和刚才一样报错了。

alter database open resetlogs
*
ERROR at line 1:
ORA-01139: RESETLOGS option only valid after an incomplete database recovery
```

### recover database until cancel
这里弄一个假的恢复骗过oracle，因为`resetlogs`要求一定是恢复完才可以使用。

```
SQL> recover database until cancel;
ORA-00279: change 1023441 generated at 03/12/2018 13:33:52 needed for thread 1
ORA-00289: suggestion : /u02/app/oracle/product/11.2.4/db1/dbs/arch1_2_636817648.dbf
ORA-00280: change 1023441 for thread 1 is in sequence #2


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
cancel
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u02/app/oracle/oradata/orcl/system01.dbf'


ORA-01112: media recovery not started
```

### 再次使用`resetlogs`选项打开数据库

```
idle>alter database open resetlogs;

Database altered.

idle>select open_mode from v$database;

OPEN_MODE
------------------------------------------------------------
READ WRITE
```

可以看到现在数据库已经被open了。

> ps:打开后，一定要full db backup。此时db是不一致的状态，所以需要将数据导出再导入。
