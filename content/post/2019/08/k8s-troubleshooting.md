
# 记录 k8s 集群的 troubleshooting 的一些方法

二进制安装的集群
master 节点需要注意的三个进程是否已经启动起来
node 节点的两个进程是否已经启动起来
如果启动不了，可以看相应的日志

master 节点`kubectl get node` 如果 timeout 了，有空能是 etcd 没有启动问题。如果要启动 etcd，又 etcd 是多块的话，应该保证同时被启动，否则很有可能其中一个节点 hang 住
node 还有 kubelet 启动不了的问题，有可能是master还没有把 etcd 拉起来，造成了数据没法读取，从而使得有些使用了分布式存储的pod 拉不起来

kubeadm 安装的集群
保证 kubelet 启动了的，因为使用的是 static pod，所以只需要保证`kubelet`进程是启动起来了即可。
