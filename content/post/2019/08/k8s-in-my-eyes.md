我眼中的kubernetes

架构

容器运行时

编排

## 和大商场类比

每一层楼 对应 namespace
每一个店 对应 pod
每一个店店大小 对应 使用磁盘的大小
每一个店的关注度 对应 使用资源
每一个店的名字 对应 deployment的名字
每一个店的特点 对应 tag

店与店的通讯 可以经过管理或者前台 对应 deployment直接的访问，跨楼层相当于跨namespace

店的升级 对应 deployment的升级
店的关闭 对应 deployment的删除

店的货品调 对应 外面去拉镜像
店的货品从仓库取 对应 本地仓库拉取镜像

特卖会 对应 istio的相关调度

促销 对应 istio的相关流量引流




引入istio

引入knative
中庭的车展 借用贵地打个广告 对应 knative


crd与operator

核心往往只是我们需要了解就可以了
组件的源码阅读

怎么去改核心代码



提高生产力的工具

