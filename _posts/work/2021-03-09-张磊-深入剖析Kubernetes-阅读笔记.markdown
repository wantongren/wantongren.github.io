---
layout: post
title: 深入剖析Kubernetes阅读笔记
date: 2021-03-09 10:21:00 +0900
categories: [work2021, k8s] 
---

本文主要记录张磊的深入剖析Kubernetes课程的一些重点和疑惑之处，以备后续品读。

---
1. Pod原地升级能力，在 Kubernetes 的默认控制器中都是不支持的。但，这是社区开源控制器项目 https://github.com/openkruise/kruise 的重要功能之一，如果你感兴趣的话可以研究一下。
2. 《Docker 容器与容器云》
3. Linux 的进程模型对于容器本身的重要意义
4. 左耳朵耗子  DOCKER基础技术：LINUX NAMESPACE（上）(下)
5. https://blog.csdn.net/gatieme/article/details/50914250
6. Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。
7. epoll实现原理
8. XtraBackup 是业界主要使用的开源 MySQL 备份和恢复工具。
9. 基于 Flannel UDP 模式的跨主通信的基本原理
![Alt_text](/public/img/work/flannel.jpg){:height="50%" width="50%" mode="widthFix"}
