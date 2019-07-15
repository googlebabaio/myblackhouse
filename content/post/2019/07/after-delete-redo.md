---
title: "删除redo后的恢复(1)"
date: 2019-07-15
draft: false
tags: ["Oracle"]
---

删除redo后的恢复步骤
<!--more-->

先说一下大体的思路，如果损坏的redo log是`INACTIVE`状态的，也就是实例崩溃恢复用不到的redo log，那处理起来比较容易，直接
```
alter database clear logfile group #;
或者
alter database clear unarchived logfile group #;
```


```
SQL> select * from v$log;----可以再次确定redo03文件不是当前日志，且已经归档

SQL>alter database clear logfile group 3;
SQL>alter database open;

但如果redo03还没归档的话，那就需要加unarchived了，
SQL>alter database clear unarchived logfile group 3;
```
重建日志组就行了。建议重建日志文件级后对数据库做一个全库备份，特别是强制clear后，造成的归档日志文件断层。

在如果损坏的redo log是`ACTIVE`或`CURRENT`状态的，也就是实例崩溃恢复需要用到的redo log，那处理起来就比较麻烦了，损坏这种redo log就意味着丢失数据。

redo log的三种状态：

- INACTIVE：日志对应的修改已经被写入硬盘
- ACTIVE：日志对应的修改还没有被写入硬盘
- CURRENT：实例正在使用的日志文件

```
idle>startup
ORACLE instance started.

Total System Global Area 1603411968 bytes
Fixed Size        2253664 bytes
Variable Size      1476398240 bytes
Database Buffers    117440512 bytes
Redo Buffers          7319552 bytes
Database mounted.
ORA-00313: open failed for members of log group 2 of thread 1
ORA-00312: online log 2 thread 1: '/u02/app/oracle/oradata/orcl/redo02.log'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

3、尝试使用clear方式重建日志组出现报错

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

从报错信息中可以看出log 2是实例崩溃恢复所需要的日志文件，不能直接重建。

4、这种情况下使用隐含参数_allow_resetlogs_corruption，创建pfile，把*._allow_resetlogs_corruption=TRUE加入到pfile中。然后mount数据库，强制不完全恢复，再open resetlogs

idle>create pfile='/home/oracle/initorcl.ora' from spfile;

File created.
[oracle@rhel6 orcl]$ vi /home/oracle/initorcl.ora
idle>shutdown immediate;
ORA-01109: database not open


Database dismounted.
ORACLE instance shut down.
idle>startup pfile='/home/oracle/initorcl.ora' mount;
ORACLE instance started.

Total System Global Area 1603411968 bytes
Fixed Size        2253664 bytes
Variable Size      1476398240 bytes
Database Buffers    117440512 bytes
Redo Buffers          7319552 bytes
Database mounted.
idle>show parameter _allow_

NAME                    TYPE                VALUE
------------------------------------ --------------------------------- ------------------------------
_allow_resetlogs_corruption      boolean                  TRUE
idle>recover database until cancel;
ORA-00279: change 1023441 generated at 02/24/2017 23:54:54 needed for thread 1
ORA-00289: suggestion : /u02/app/oracle/product/11.2.4/db1/dbs/arch1_2_936817668.dbf
ORA-00280: change 1023441 for thread 1 is in sequence #2


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
cancel
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u02/app/oracle/oradata/orcl/system01.dbf'


ORA-01112: media recovery not started


idle>alter database open resetlogs;

Database altered.

idle>select open_mode from v$database;

OPEN_MODE
------------------------------------------------------------
READ WRITE

可以看到现在数据库已经被open了。

5、再次查看第一步中被删除的数据的表，数据仍然存在说明丢失CURRENT或ACTIVE状态的日志文件会导致数据丢失。

idle>select count(*) from zx;

  COUNT(*)
----------
      2858

以上是在虚拟机上做测试的恢复过程，但是对于前面说到的开发库的恢复就没有这个过程简单了。可以说是解决了一个报错又出来新的报错。

在使用_allow_resetlogs_corruption参数执行不完全恢复，open resetlogs 时，遇到了ORA-01248

SQL> alter database open resetlogs;
```


```
对于当前redo的故障，基本就是丢失这个redo的数据了，闲杂主要就是用备份将数据库恢复到最近的状态，肯定是不完全恢复了。要不就是强制将db拉起来。
用备份恢复，这里就不写了，基本就是restore，recover until cancel的。现在做一下强制拉起来的情况：
SQL> alter database clear unarchived logfile group 1;--尝试clear，失败
alter database clear unarchived logfile group 1
*
ERROR at line 1:
ORA-01624: log 1 needed for crash recovery of instance dong (thread 1)
ORA-00312: online log 1 thread 1: '/u01/app/oracle/oradata/dong/redo01.log'


SQL> shutdown abort
ORACLE instance shut down.
通过修改隐含参数来启动：
将*._allow_resetlogs_corruption=TRUE加入到参数文件。
[oracle@baobao dbs]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.1.0 Production on Thu Nov 21 16:47:57 2013

Copyright (c) 1982, 2009, Oracle.  All rights reserved.

Connected to an idle instance.

SQL> startup mount pfile='$ORACLE_HOME/dbs/initdong.ora';
ORACLE instance started.

Total System Global Area 2092498944 bytes
Fixed Size                  1337604 bytes
Variable Size             268437244 bytes
Database Buffers         1811939328 bytes
Redo Buffers               10784768 bytes
Database mounted.
SQL> alter database open resetlogs;--这里一定要用resetlogs呀，因为你要是不用的话，就是代表要用redo进行实例恢复了，那肯定会和刚才一样报错了。
alter database open resetlogs
*
ERROR at line 1:
ORA-01139: RESETLOGS option only valid after an incomplete database recovery


SQL> recover database until cancel;--这里我们弄一个假的恢复骗过oracle，因为resetlogs要求一定是恢复完才可以使用。
Media recovery complete.
SQL> alter database open resetlogs;--打开db，完毕

Database altered.

打开后，一定要full db backup。此时db是不一致的状态，所以需要将数据导出再导入。
```
