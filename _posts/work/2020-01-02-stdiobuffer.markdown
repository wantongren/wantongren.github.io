---
layout: post
title: stdio buffer--k8s日志乱序问题总结
date: 2021-01-07 21:20:23 +0900
categories: [work2021, stdio, k8s]
---
## 问题背景
---
pipeline输入日志中错误日志显示的位置异常，早于产生错误的命令调用，如下图：

![Alt text](/_posts/work/resource/日志乱序.png)

我们发现2处的命令错误输出在1处显示，这个顺序是错乱的。

## 问题原因
---
这里的日志是从标准输出( stdout )和标准错误( stderr )中收集的，当这两类日志重定向到文件时，默认stdout是全缓存模式（缓存大小为4k，页大小）， 默认stderr是无缓存的，这就导致了日志收集时错误日志会先于某些标准输出日志先被收集。

## 解决方案
---
1. stdbuf 命令， 将标准输出模式改为行缓存。
```ruby
      function tmpf() { echo aaa; }
      export -f tmpf
      stdbuf -o 0 bash -c 'tmpf'
```
2. 使用环境变量 BUF_X_=Y  ，其中 X = 0 (stdin), 1 (stdout), 2 (stderr)， Y = 0 (unbuffered), 1 (line buffered), >1 buffered + size。 
     
     > 例如： tail -f access.log | BUF_1_=1 cut -d' ' -f1 | uniq
     
     该方法经测试无效。
     
## 资料总结
---
缓冲类型分为三种：
* 无缓冲
* 行缓冲
* 全缓冲

stderr默认缓冲就是无缓冲。而stdout的缓冲类型与输出介质有关：
- 屏幕或者终端：行缓冲
- 重定向文件、管道：全缓冲

一般情况下程序输出介质都是屏幕或者终端,采用的都是行缓冲，也就是实时输出。但是当程序输出介质为重定向文件或者管道时，内核为了性能优化，可能变成非实时的。究其原因也就是因为pipe的缓冲区问题。

> 例子
>   
>   tail -f access.log | cut -d' ' -f1 | uniq   

发现没有任何输出。

![Alt text](/_posts/work/resource/stdio.png)

分析:
首先tail程序通过read调用读取文件中的内容,然后将读取的数据写入stdout,由于tail的输出是管道，需要拷贝到内核。（tail程序比较特殊,在有新数据产生时，会主动调用fflush，刷新缓冲区。），内核中第一个缓冲区如果不写满，cut程序便读取不到tail写入的数据（原因1）。由于cut程序本身也是缓冲的（原因2），输出是管道，这里也会等待缓冲区满（原因3），uniq程序才能读取到数据。最终导致了没有任何输出。
类似的程序还有 tcpdump -l ，grep --line-buffered , sed --unbuffered 提供了参数使其变成非缓冲的。
方案:
知道原因后，其实也就是告诉程序我不要缓冲,实时给我写就好。linux提供了stdbuf命令。
```
stdbuf -oL command
tail -f ~/access.log | stdbuf -oL cut -d' ' -f1 | uniq
```
其中的参数，o表示输出流，L表示行缓冲。 这样主要遇到换行符，就会将缓冲输出到指定对象。而不会等到缓冲区完全写满后，才让下游读取。

参考链接： [buffering in standard streams](http://www.pixelbeat.org/programming/stdio_buffering/)