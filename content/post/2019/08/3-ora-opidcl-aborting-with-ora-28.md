查询一下scanip下注册的服务 有没有包括 sessoin数少的那个节点的监听信息

```
lsnrctl status listener_scan1
```

然后检查一下两个节点的remote_listener

1、两个实例的内存消耗，差异，具有又差多少呢？

其他：

1、你看看内存消耗比较高的实例，会话数是否高于另外一个实例。
    select  inst_id,count(*) from gv$session;
2、负载均衡的效果不平均，主要受限于以下两个因素：
   A、连接风暴，很短的时间内发出大量的连接请求。
   B、系统负载过高。
   C、从理论上来说，RAC运行时间越长，LB的效果会更好。

这个是 alert log 中记录异常退出的进程

最后那个ora-28代表是session被kill了



可以参考：
https://www.cnblogs.com/bicewow/p/10729632.html
https://www.cnblogs.com/tianlesoftware/archive/2012/08/04/3609196.html
https://blog.csdn.net/tianlesoftware/article/details/7829166/


* https://blog.csdn.net/weixin_33829657/article/details/92499772