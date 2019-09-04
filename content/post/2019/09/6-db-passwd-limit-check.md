1.还在使用默认密码的用户
```
select * from DBA_USERS_WITH_DEFPWD
```

弱口令需要自查

2.启用登录失败处理功能

查看视图`dba_profiles`可找出数据库中有哪些`PROFILE`
```
select distinct profile from dba_profiles;
```

(1)对数据库资源做限制
```
{ { SESSIONS_PER_USER 每个用户名所允许的并行会话数
  | CPU_PER_SESSION   一个会话一共可以使用的CPU时间，单位是百分之一秒
  | CPU_PER_CALL      一次SQL调用(解析、执行和获取)允许使用的CPU时间
  | CONNECT_TIME      限制会话连接时间，单位是分钟
  | IDLE_TIME         允许空闲会话的时间，单位是分钟
  | LOGICAL_READS_PER_SESSION 限制会话对数据块的读取，单位是块
  | LOGICAL_READS_PER_CALL    限制SQL调用对数据块的读取，单位是块
  | COMPOSITE_LIMIT   “组合打法”
  }   { integer | UNLIMITED | DEFAULT }
  | PRIVATE_SGA   限制会话在SGA中Shared Pool中私有空间的分配  { size_clause | UNLIMITED | DEFAULT}
}
```

(2)对密码做限制
```
{ { FAILED_LOGIN_ATTEMPTS 帐户被锁定之前可以错误尝试的次数
  | PASSWORD_LIFE_TIME    密码可以被使用的天数，单位是天，默认值180天
  | PASSWORD_REUSE_TIME   密码可重用的间隔时间(结合PASSWORD_REUSE_MAX)
  | PASSWORD_REUSE_MAX    密码的最大改变次数(结合PASSWORD_REUSE_TIME)
  | PASSWORD_LOCK_TIME    超过错误尝试次数后，用户被锁定的天数，默认1天
  | PASSWORD_GRACE_TIME   当密码过期之后还有多少天可以使用原密码
  }  { expr | UNLIMITED | DEFAULT }
  | PASSWORD_VERIFY_FUNCTION  { function | NULL | DEFAULT }
}
```

修改profile：
```
alter profile [资源文件名] limit [资源名] unlimited;
```
如：
```
alter profile default limit failed_login_attempts 100;
```

3.查询`seelct * from dba_users`
锁定不使用的用户 ```alter user xxx account lock;```
删除不使用的业务账号 ```drop user xxx cascade;```

> ps: MySQL查询 mysql.users这个视图

4.使用特定的ip对数据库进行访问
```
# su - oracle
$ cd $ORACLE_HOME/network/admin/
$ vi sqlnet.ora

sqlnet.ora中进行下列参数的设置可以限制或允许用户从特定的客户机连接到数据库中。
tcp.validnode_checking=yes|no
tcp.invited_nodes=(ip|hostname,...)
tcp.excluded_nodes=(ip|hostname,...)
##如果是hostname 则需要在/etc/hosts 里面配置对应的ip
tcp.validnode_checking   参数确定是否对客户机IP地址进行检查；
tcp.invited_nodes        参数列举允许连接的客户机的IP地址；
tcp.excluded_nodes       参数列举不允许连接的客户机的IP地址。
```

需要注意的地方：
(1)tcp.invited_nodes与tcp.excluded_nodes都存在，以tcp.invited_nodes为主
(2)一定要许可或不要禁止服务器本机的IP地址，否则通过lsnrctl将不能启动或停止监听，因为该过程监听程序会通过本机的IP访问监听器，而该IP被禁止了，但是通过服务启动或关闭则不影响。
(3)修改之后，分两种情况
  如果是第一次使用sqlnet.ora 文件，则需要重启数据库。
  如果之前已经使用了sqlnet.ora 则不需要重启数据库，reload 监听就可以！
(4)任何平台都可以，但是只适用于TCP/IP协议
