---
layout: post
title: Kubernetes 应用性能分析工具-Kubectl Flame
date: 2021-02-18 17:33:00 +0900
categories: [work2021, k8s] 
---

Profile 是分析应用程序性能来改进代码质量的常用方法，最流行的可视化性能分析方法是生成火焰图。


## Kubernetes 上的性能分析

性能分析是一项较为复杂的任务，大多数探查器有两个主要问题：

* 需要修改应用程序，通常可以通过将标志添加到执行命令或将一些性能分析库导入代码中来实现。
* 由于在分析过程中会严重影响性能，因此通常避免在生产环境中进行性能分析。

选择正确的探查器可能会解决这些问题，但是这需要仔细去进行研究，并且通常取决于编程语言和操作系统。

在 Kubernetes 集群中运行的应用程序上执行分析时，会变得更加困难。需要部署一个包含配置文件修改的新容器镜像，而不是当前正在运行的容器。此外，当应用程序重新启动时，某些性能问题可能会消失，这就使得调试变得困难。

## kubectl flame
 
Kubectl Flame 是一个 kubectl 插件，可以以较低的开销生成火焰图🔥来分析应用程序性能，无需进行任何应用程序修改或停机。

#### 安装

可以通过 Krew 来安装 **kubectl flame** 插件，一旦安装了Krew，就可以通过如下命令进行安装：

    `kubectl krew install flame`

#### 运行原理

kubectl-flame 通过在与目标容器相同的节点上启动一个探查器来启动性能分析，大多数探查器将与目标容器共享一些资源：比如通过将 **hostPID** 设置为 true 来启用 PID 命名空间共享，通过挂载 **/var/lib/docker** 并查询 overlayFS 来启用文件系统共享。在后台kubectl-flame使用 async-profiler 来为 Java 应用程序生成火焰图，通过共享 **/tmp** 文件夹与目标 JVM 进行交互，Golang 则支持基于 ebpf 分析，Python 支持基于 py-spy 进行分析。

![Alt_text](/public/img/work/k8s-flame.jpg){:height="50%" width="50%" mode="widthFix"}

#### 分析 Kubernetes Pod

分析 Java 应用 mypod 1分钟，并在将火焰图保存到 /tmp/flamegraph.svg：

    `kubectl flame mypod -t 1m --lang java -f /tmp/flamegraph.svg`
  
#### 分析基于 alpine 的容器

在基于 alpine 的容器中分析 Java 应用程序需要使用 **--alpine** 标志：

    `kubectl flame mypod -t 1m -f /tmp/flamegraph.svg --lang Java --alpine`
  
  **注意：仅 Java 应用程序需要此--alpine标志，而 Go 分析则不需要该标志。**
  
#### 分析 sidecar 容器
 
包含多个容器的 Pod 需要将目标容器指定为参数： 
 
    `kubectl flame mypod -t 1m --lang go -f /tmp/flamegraph.svg mycontainer`
   
   **源码仓库地址：https://github.com/VerizonMedia/kubectl-flame**