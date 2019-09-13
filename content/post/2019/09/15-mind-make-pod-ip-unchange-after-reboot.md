
1. 创建自定义网络(指定ip需要重启docker服务)
```
docker network create mynet && service docker restart
```
Docker支持对自定义网络使用--ip制定自定义网络容器ip地址，如果未制定--ip参数（即首次创建容器会自动分配一个）。

2. 创建脚本文件 recreateContainer.sh

```
#!/bin/sh
export DOCKER_NETWORK=mynet
export DOCKER_REGISTRY=172.16.1.160:5000
ip=`docker inspect $1 | jq .[].NetworkSettings.Networks.$DOCKER_NETWORK.IPAddress`
if [ "$ip" = '""' ]
then
        #container stoped,cannot get ip from docker inspect, try to get ip from hosts
        hostname=`docker inspect $1 | jq .[].Config.Hostname|sed 's/\"//g'`
        id=`docker inspect $hostname | jq .[].Id | sed 's/\"//g' `
        ip=`cat /var/lib/docker/containers/$id/hosts| grep $hostname | awk '{print $1}'`
fi

#replace "
ip=`echo $ip |sed 's/\"//g'`
docker rm -f $1
if [ "$ip" = '' ]
then
        docker run -d --restart=always --name=$1 --hostname=$1 --network=$DOCKER_NETWORK $DOCKER_REGISTRY/centos /bin/tail -f /etc/hosts
else
        docker run -d --restart=always --name=$1 --hostname=$1 --network=$DOCKER_NETWORK --ip=$ip $DOCKER_REGISTRY/centos /bin/tail -f /etc/hosts
fi
```





3. 安装json解析工具jq
```
# sudo dnf install -y jq
```

4. 测试脚本
```
# sh recreateContainer.sh os1
os1
60dc6c04b6be130a4eff047c6d64367025ef34c96d78e55134fc44efc2295979
```

5.参考一下
https://blog.csdn.net/sannerlittle/article/details/77063800
