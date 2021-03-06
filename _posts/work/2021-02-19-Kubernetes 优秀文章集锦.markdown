---
layout: post
title: Kubernetes优秀文章集锦 
date: 2021-02-19 14:38:00 +0900
categories: [work2021, k8s] 
---

本文主要记录日常查阅资料中找到的一些优秀的k8s相关的文章，以备后续品读。

---

1. [宝树呐--随笔分类--Kubernetes](https://www.cnblogs.com/baoshu/category/1492542.html)

    该链接中的文章清晰的讲解了k8s集群调度、存储相关、网络、Pod相关、Service、资源清单、Ingress、Helm等内容，逻辑清楚，举例恰当。
    * kubernetes系列(十一) - 存储之configMap : configMap的热更新只有对volume挂载文件的使用方式有效，对Env无效。
    * kubernetes系列(十) - 通过Ingress实现七层代理 : 
        * 边缘节点Edge Node，所谓的边缘节点即集群内部用来向集群外暴露服务能力的节点，集群外部的服务通过该节点来调用集群内部的服务，边缘节点是集群内外交流的一个Endpoint
        * Keepalived 是运行在 lvs 之上，是一个用于做双机热备（HA）的软件，它的主要功能是实现真实机的故障隔离及负载均衡器间的失败切换，提高系统的可用性。
    * kubernetes系列(七) - Pod生命周期  :
        * 对Init容器spec的修改被限制在容器image字段，修改其他字段都不会生效。更改Init容器的image字段，等价于重启该Pod 
        * ReadinessProbe: 用于判断容器服务是否可用，成功的话pod状态会显示成ready，失败的话只是不显示ready，不会触发重启策略
        * LiveinessProbe: 用于判断运行中的容器是否存活，如果探测到不健康，则kubelet会杀掉该容器，并触发重启策略。如果容器没有配置liveinessProbe，则kubelet认为该容器的liveinessProbe返回的永远是succeed     