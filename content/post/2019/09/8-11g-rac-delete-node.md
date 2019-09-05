---
title: "11gRAC删除节点步骤"
subtitile: ""
date: 2019-06-19
draft: false
tags: ["k8s"]
---

给出11gRAC删除节点步骤，以供参考
<!--more-->

> 参考
http://docs.oracle.com/cd/E11882_01/rac.112/e41959/adddelclusterware.htm#CWADD90989

1、使用root用户检查节点是否被占用（pinned），ORACLE_HOME=/u01/app/11.2.0/grid

```
[root@host01 ~]# olsnodes -s -t
host01	Active	Unpinned
host02	Active	Unpinned
host03	Active	Unpinned
```

2、在需要删除的节点host03目录ORACLE_HOME/crs/install下使用root执行 ORACLE_HOME=/u01/app/11.2.0/grid
```
[root@host03 ~]# $ORACLE_HOME/crs/install/rootcrs.pl -deconfig -force
Using configuration parameter file: /u01/app/11.2.0/grid/crs/install/crsconfig_params
PRCR-1119 : Failed to look up CRS resources of ora.cluster_vip_net1.type type
PRCR-1068 : Failed to query resources
Cannot communicate with crsd
PRCR-1070 : Failed to check if resource ora.gsd is registered
Cannot communicate with crsd
PRCR-1070 : Failed to check if resource ora.ons is registered
Cannot communicate with crsd

CRS-4544: Unable to connect to OHAS
CRS-4000: Command Stop failed, or completed with errors.
Removing Trace File Analyzer
Successfully deconfigured Oracle clusterware stack on this node
```

> 删除时一定要看清楚是哪一台!

3、在任意一台未删除的节点使用root执行如下命令
```
[root@host02 ~]# crsctl delete node -n host03
CRS-4661: Node host03 successfully deleted.
```

4.检查集群是否已经删除，这里可以看到host03已经全部删除了

```
[root@host02 ~]# crs_stat -t
Name           Type           Target    State     Host
------------------------------------------------------------
ora.DATA.dg    ora....up.type ONLINE    ONLINE    host02
ora.FRA.dg     ora....up.type ONLINE    ONLINE    host02
ora....ER.lsnr ora....er.type ONLINE    ONLINE    host02
ora....N1.lsnr ora....er.type ONLINE    ONLINE    host02
ora....N2.lsnr ora....er.type ONLINE    ONLINE    host01
ora....N3.lsnr ora....er.type ONLINE    ONLINE    host02
ora.asm        ora.asm.type   ONLINE    ONLINE    host02
ora.cvu        ora.cvu.type   ONLINE    ONLINE    host02
ora.gsd        ora.gsd.type   OFFLINE   OFFLINE
ora....SM2.asm application    ONLINE    ONLINE    host02
ora....02.lsnr application    ONLINE    ONLINE    host02
ora.host02.gsd application    OFFLINE   OFFLINE
ora.host02.ons application    ONLINE    ONLINE    host02
ora.host02.vip ora....t1.type ONLINE    ONLINE    host02
ora....SM3.asm application    ONLINE    ONLINE    host01
ora....01.lsnr application    ONLINE    ONLINE    host01
ora.host01.gsd application    OFFLINE   OFFLINE
ora.host01.ons application    ONLINE    ONLINE    host01
ora.host01.vip ora....t1.type ONLINE    ONLINE    host01
ora....network ora....rk.type ONLINE    ONLINE    host02
ora.oc4j       ora.oc4j.type  ONLINE    ONLINE    host01
ora.ons        ora.ons.type   ONLINE    ONLINE    host02
ora.racdb.db   ora....se.type OFFLINE   OFFLINE
ora.scan1.vip  ora....ip.type ONLINE    ONLINE    host02
ora.scan2.vip  ora....ip.type ONLINE    ONLINE    host01
ora.scan3.vip  ora....ip.type ONLINE    ONLINE    host02
```

5、在任意台未被删除节点host01和host02上均使用grid和oracle分别执行
```
[grid@host02 ~]$ $ORACLE_HOME/oui/bin/runInstaller -updateNodeList ORACLE_HOME=$ORACLE_HOME "CLUSTER_NODES={host03}" CRS=TRUE -silent -local
Starting Oracle Universal Installer...

Checking swap space: must be greater than 500 MB.   Actual 3957 MB    Passed
The inventory pointer is located at /etc/oraInst.loc
The inventory is located at /u01/app/oraInventory
'UpdateNodeList' failed.
[grid@host02 ~]$ $ORACLE_HOME/oui/bin/runInstaller -updateNodeList ORACLE_HOME=$ORACLE_HOME "CLUSTER_NODES={host02,host01}" CRS=TRUE -silent -local
Starting Oracle Universal Installer...

Checking swap space: must be greater than 500 MB.   Actual 3957 MB    Passed
The inventory pointer is located at /etc/oraInst.loc
The inventory is located at /u01/app/oraInventory
'UpdateNodeList' was successful.
[oracle@host02 ~]$ $ORACLE_HOME/oui/bin/runInstaller -updateNodeList ORACLE_HOME=$ORACLE_HOME "CLUSTER_NODES={host02,host01}"
Starting Oracle Universal Installer...

Checking swap space: must be greater than 500 MB.   Actual 3957 MB    Passed
The inventory pointer is located at /etc/oraInst.loc
The inventory is located at /u01/app/oraInventory
'UpdateNodeList' was successful.
```

6、检查是否完全从集群移除，分别用grid和oracle检查
```
[grid@host02 ~]$ cluvfy stage -post nodedel -n host03 -verbose

Performing post-checks for node removal

Checking CRS integrity...

Clusterware version consistency passed
The Oracle Clusterware is healthy on node "host02"

CRS integrity check passed
Result:
Node removal check passed

Post-check for node removal was successful.
[oracle@host02 ~]$ cluvfy stage -post nodedel -n host03 -verbose

Performing post-checks for node removal

Checking CRS integrity...

Clusterware version consistency passed
The Oracle Clusterware is healthy on node "host01"
The Oracle Clusterware is healthy on node "host02"

CRS integrity check passed
Result:
Node removal check passed

Post-check for node removal was successful.
```

7.剩余主机上应用的删除工作$ORACLE_HOME/deinstall/deinstall –local 使用各自的oracle和grid用户进行删除即可。
