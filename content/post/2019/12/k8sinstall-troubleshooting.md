搭建Kubernetes集群踩坑日志之coreDNS 组件出现CrashLoopBackOff问题的解决
https://blog.csdn.net/u011663005/article/details/87937800


报错信息:
```
tcp 10.96.0.1:443 : connect : network is unreceachable
```

> 这个方法还没有试,下周试试
> https://blog.csdn.net/shida_csdn/article/details/102612205

flannel也有一个网卡报错的问题
```
flannel failed to find any valid interface
```

解决办法:
flannel也有一个网卡报错的问题
https://www.linuxba.com/archives/8148

Kubernetes - flannel的默认网卡设置
https://blog.csdn.net/qingyafan/article/details/93519196
