Redo丢失场景和处理方法




type of Failure   | Status Column of V$LOG  |  Action
--|---|--
One member failed in multiplexed group  | N/A  |  Re-create member
All members of group	  | INACTIVE  |  Clear logfile
All members of group	  | ACTIVE | Attempt checkpoint,and if successful, clearlogfile.If checkpoint is unsuccessful, perform incomplete recovery|
All members of group	  | CURRENT  |  Attempt to clear log,if unsuccessful, perform incomplete recovery



v$log和v$logfile视图中，都有status列，不过二者有不同的含义：

- v$log中反映log group的状态；
- v$logfile中反映物理的online redo log的状态。



v$log视图中status列说明

- status	说明
- CURRENT	日志组正在被lgwr写入
- ACTIVE	crash recovery需要该日志组，可能已经被归档或者尚未被归档
- CLEARING	日志组被alter database clear logfile.. 命令清理中
- CLEARING_CURRENT	关闭的thread正在清理该日志组
- INACTIVE	crash recovery不再需要该日志组。可能已经被归档或者尚未归档
- UNUSED	最近创建尚未被使用



v$logfile视图中status列说明

- status	说明
- INVALID	该日志文件成员不可访问，或最近刚创建
- DELETED	该日志文件成员不再使用
- STALE	该日志文件成员内容不完整
- NULL	该日志文件成员正在被数据库使用


## Restoring After Losing One Member of the Multiplexed Group

1.找出media failure的online redo log
2.确保发生failure的log不是在current online log group
3.删除受损的日志成员
```
SQL> alter database drop logfile member '/u11/app/oracle/oradata/ora11/redo03b.log';
```
4.增加新的日志组成员
```
SQL> alter database add logfile member '/u11/app/oracle/oradata/ora11/redo03b.log' to group 3;
```

## Recovering After Loss of All Members of the INACTIVE Redo Log Group

- 1.找出media failure的online redo log group
- 2.确保发生failure的日志组是inactive状态
- 3.使用clear logfile命令重建日志组

  ```
  SQL> alter database clear logfile group 3;
  如果损坏的日志组没有被归档，需要添加关键字unarchive SQL> alter database clear unarchived logfile group 3;
  ```
- 4.如果损坏的日志组没有被归档，建议立即备份数据库



Recovering After Loss of All Members of the ACTIVE Redo Log Group

- 1.找出media failure的online redo log group
- 2.确保发生failure的日志组是active状态
- 3.尝试发生一个检查点
- 4.如果检查点成功，状态会变成inactive状态，然后使用clear logfile命令重建日志组
- 5.如果被clear的日志组没有归档，建议备份数据库
- 6.如果4失败，需要进行不完全恢复
