---
title: "ADG迁移经验分享"
date: 2019-07-19
draft: false
tags: ["Oracle"]
---


最近做了几个地方的双活，遇到的情况几乎都是要将以前老版本的10g迁移到11g上，并搭建双活。
方便今后参考，特总结一下
<!--more-->


## 在时间窗口比较大的情况下，建议步骤：

1. 可以提前安装主备的RAC环境
2. 在两套RAC上搭建双活
3. 在10g的环境做expdp
4. 在11g的主库RAC上做impdp
5. 在11g的主库上做utlrp编译，以及收集统计信息
6. 验证


## 经验分享
### 1.巧妙的利用dbca来做备库的配置
做RAC到RAC的双活的时候，为了简化后期的操作步骤，已经在备库创建好了同主库数据库名一致的实例。创建实例时指定db_name、db_unique_name、service_name。在配置过程中只是删除物理文件，其他资源保留。

### 2.数据泵在10g的导出脚本参考
```
[oracle@tmp]$ cat exp_orcl0823.sh
export ORACLE_SID=orcl
nohup expdp \'/ as sysdba\' parallel=4 DIRECTORY=dump_dir DUMPFILE=orcl0823_%U.dmp LOGFILE=orcl0823.log full=y job_name=orcl0823_expdp exclude=statistics &
```

### 3.导出之前的操作
为了防止新的连接进来，造成数据的不一致。所以可以先把监听停了，然后`kill sesion`

10g需要在两个节点分别做
```
alter system kill session 'sid,serial#';
```


11g只需要在一个节点做
```
alter system kill session 'sid,serial#,@inst_id';
```

### 4.在导入之前需要做的准备工作

1. 创建相应的表空间和临时表空间
2. 相关的用户不需要创建，在imdp的时候用户/密码/权限都会被相应的创建出来
3. 查询一下用户/用户表/表空间的所属，看是否存在权限的问题。遇到过一个用户在某些表空间没权限，但它的表却有存放在这个表空间。用 `alter user user_name quota tablespace_name` 来搞定


### 5.数据泵在11g的导入脚本参考
```
[oracle@temp]$ more impdp_orcl0823.sh
export ORACLE_SID=orcl
nohup impdp  \'/ as sysdba\' DIRECTORY=dump_dir parallel=4 dumpfile=orcl0823_%U.dmp logfile=impdp_orcl0823.log SCHEMAS=scott,hr,sh cluster=no job_name=impdp_orcl0823 &
```

### 6.补充导入数据，如同义词
在双活的部署过程中，发现连接备库的app启动时报表不存在，经排查发现是备库对应的用户使用了同义词去访问同一个库的其他用户的表，但是expdp的时候选用full=y的不会单独备份同义词的。后来经过手工创建同义词搞定
```
SELECT 'create public synonym ' || synonym_name || ' for ' || table_owner || '.' || table_name || ';'
  FROM dba_synonyms s WHERE s.owner = 'PUBLIC' AND s.table_owner = UPPER ('&input_owner');
```

当然我们也可以在expdp的时候把同义词也导出来：
```
expdp sys/Oracle directory=dump_dir dumpfile=syns.dmp logfile=exp_syns.log full=y include=PUBLIC_SYNONYM/SYNONYM:\"IN \(SELECT synonym_name FROM dba_synonyms WHERE table_owner=\'USER_NAME\'\)\"
```

再导入进去：
```
impdp sys/Oracle directory=dump_dir dumpfile=syns.dmp logfile=imp_syns.log full=y include=synonym
```

include具体怎么写，可以查询数据字典：
`select * from database_export_objects t` 的`path`那一列

### 7.特别注意dg对于临时文件的同步有点问题
在主库上创建的临时表空间，在备库有定义，但是查询gv$tempfile查不到文件。这个也是日了狗了。
可以采用两种办法解决：

- 方法1：在主库上给相应的临时表空间添加数据文件
  ```
  alter tablespace xxxx add tempfile '+DATADG' size  10g autoextend on;
  ```

- 方法2：将用户的默认临时表空间指向temp
  ```
  alter user xxx default temporary tablespace temp;
  ```
