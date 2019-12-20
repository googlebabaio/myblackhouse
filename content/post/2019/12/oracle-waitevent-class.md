# Oracle等待事件种类

1. administrative(管理)
   由于DBA命令导致的等待， 如索引的重建
2. application
  由于应用原因导致的等待。如行级锁等待或明确的锁命令（for update应该可以算其中一种）
3. 集群
 与Oracle Real Application Clusters资源有关的等待（如global cache resources such as 'gc cr block busy'）
4. commit
  这个等待事件类型只包含一种等待事件-wait for redo log write confirmation after a commit(log file sync)
5. Concurrency
  等待内部数据库资源（如，latches）
6. Configuration
  Waits caused by inadequate configuration of database or instance resources(如undersized log file sizes, share pool sizes)
 简单来说就是由于数据库或者实例资源配置，如过小的redo日志文件，过小的共享池导致的等待事件
7. Idle
 空闲等待。 Waits that signify the session is inactive, waiting for work(for example, 'SQL*Net message from client')
 由于会话处于非活动状态，等待工作信号？
8. Network
  与网络消息有关的等待， 如SQL*Net more data to dblink
9. Other
 Waits which should not typically occur on a system(如，'wait for Emon to spawn')
10. Queue
 Contains events that signify delays in obtaining additional data in a pipelined environment. The time spent on these wait evens indicates inefficiency or other problems
in the pipeline, It affects features such as Oracle Streams, parallel queries,  or DBMS_PIPE PL/SQL packages.
11. Scheduler
  Resource Manager related waits(for example, 'resmgr: become active')
12. System I/O
 Waits for background process I/O, for example DBWR wait for 'db file parallel write'
13. User I/O
 Waits for user I/O, for example db file sequential read
