---
title: "设置Oracle开机自动启动"
date: 2019-07-18
draft: false
tags: ["Oracle"]
---

记录一下Oracle开机自动启动步骤
<!--more-->

1.修改dbstart脚本
修改ORACLE_HOME/bin/dbstart脚本中的`ORACLE_HOME_LISTNER=ORACLEH​OME/bin/dbstart`脚本中的`ORACLEH​OMEL​ISTNER=ORACLE_HOME`

2.修改 /etc/oratab 文件
修改 /etc/oratab ，将
```
orcl:/u01/app/oracle/product/11.2.0/dbhome_1:N
```
修改为:
```
orcl:/u01/app/oracle/product/11.2.0/dbhome_1:Y
```

3.修改 /etc/rc.d/rc.local 文件
修改/etc/rc.d/rc.local启动文件，添加数据库启动脚本dbstart
```
touch /var/lock/subsys/local
su oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/lsnrctl start"
su oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/dbstart"
```

>注意:
在这个rc.local中的touch /var/lock/subsys/local
它的作用是判断是否已经执行过 rc.local ,如果已经执行过则会建立一个/var/lock/subsys/local文件,否则就回去执行这个/etc/rc.d/rc.local
另外，之前遇到过修改了rc.local脚本但不执行的问题,原因是rc.local文件的权限问题

4.重启服务器验证
