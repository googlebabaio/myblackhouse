搭建Kubernetes集群踩坑日志之coreDNS 组件出现CrashLoopBackOff问题的解决
https://blog.csdn.net/u011663005/article/details/87937800


报错信息:
```
tcp 10.96.0.1:443 : connect : network is unreceachable
```

flannel也有一个网卡报错的问题
```
flannel failed to find any valid interface
```

解决办法:
flannel也有一个网卡报错的问题
https://www.linuxba.com/archives/8148

Kubernetes - flannel的默认网卡设置
https://blog.csdn.net/qingyafan/article/details/93519196
