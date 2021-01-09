---
layout: post
title: k8s orphan pod问题及解决方案
date: 2020-01-09 14:20:23 +0900
categories: [work2021, k8s] 
---
## 问题描述
k8s集群中node节点日志`（/var/log/message）`输出大量错误日志：
>
>orphaned pod "002d97dc-e398-4e34-b510-38f52354ad29" found, but volume paths are still present on disk : There were a total of 2044 errors similar to this. Turn up verbosity to see them.
>


## 原因分析
&emsp;&emsp;Pod 异常退出，导致数据卷挂载点在卸载过程中没有清理干净，最终导致Pod沦为僵尸Pod。Kubelet的GC流程对数据卷垃圾回收实现并不完善，目前需要手动或脚本自动化实现垃圾挂载点的清理工作。
&emsp;&emsp;该节点上的pod（该pod挂载了volume）被delete后，相应的pod路径没有清理，导致kubelet认为该pod还存在，实际pod已经没有了。

## 解决方案
* **方法1**：
    
    kubelet源码中进行修改，清理对应的路径，现在已经提交了pr，但是仍处于为合并状态。
    社区讨论区：[kubernetes社区讨论区](https://github.com/kubernetes/kubernetes/issues/60987)

* **方法2**：
    
    在每个node上定期执行清理脚本。
````shell
    #!/bin/bash
    #
    # This script is designed to be run as a cron job periodically to
    # clean up the Orphaned Pods. Use at your own risk!
    #
    
    ## Settings (can be passed as environment variables)
    
    LOGFILE=${LOGFILE:-/var/log/messages} # what log file to process
    KUBELET_PODS_DIR=${KUBELET_DIR:-/media/ssd1/kubelet/pods} # where to find pods to remove
    DEBUG=${DEBUG:-1} # more debug output
    echo ${LOGFILE}
    echo ${DEBUG}
    
    
    ## Script Starts Here
    last_pod=""
    
    # Retrieve the most recent "Orphaned pod" log entry (tac =  backwards cat)
    # NOTE: If there are no orphaned pods, then pod==last_pod, so its skips :)
    while pod=$(tail ${LOGFILE} | grep -m 1 'orphaned pod' | grep -o '".*"' | sed 's/"//g'); do
      echo "pod:"${pod}
      last_pod=$pod
      pod_dir="${KUBELET_PODS_DIR}/${pod}"
      if [[ ${pod} != "" ]]; then 
         mount | grep ${pod_dir}
        if [[ $? == 0 ]] && [[ -d $pod_dir ]]; then
          echo "rm file: "${pod_dir}
          #rm -rf "${pod_dir}"
          if [[ $? == 0 ]]; then
            echo "Cleaned orphaned pod volumes: $pod"
          else
            echo "ERROR: Pod cannot be deleted: ${pod_dir}" >&2
          fi
        else
          [[ $DEBUG == 1 ]] && echo "DEBUG: $pod doesn't exist in directory (${pod_dir})"
        fi
      else
        [[ $DEBUG == 1 ]] && echo "DEBUG: no orphan pod to delete"
      fi 
      sleep 3 
      echo 
    done#!/bin/bash
        #
        # This script is designed to be run as a cron job periodically to
        # clean up the Orphaned Pods. Use at your own risk!
        #
        
        ## Settings (can be passed as environment variables)
        
        LOGFILE=${LOGFILE:-/var/log/messages} # what log file to process
        KUBELET_PODS_DIR=${KUBELET_DIR:-/media/ssd1/kubelet/pods} # where to find pods to remove
        DEBUG=${DEBUG:-1} # more debug output
        echo ${LOGFILE}
        echo ${DEBUG}
        
        
        ## Script Starts Here
        last_pod=""
        
        # Retrieve the most recent "Orphaned pod" log entry (tac =  backwards cat)
        # NOTE: If there are no orphaned pods, then pod==last_pod, so its skips :)
        while pod=$(tail ${LOGFILE} | grep -m 1 'orphaned pod' | grep -o '".*"' | sed 's/"//g'); do
          echo "pod:"${pod}
          last_pod=$pod
          pod_dir="${KUBELET_PODS_DIR}/${pod}"
          if [[ ${pod} != "" ]]; then 
             mount | grep ${pod_dir}
            if [[ $? == 0 ]] && [[ -d $pod_dir ]]; then
              echo "rm file: "${pod_dir}
              #rm -rf "${pod_dir}"
              if [[ $? == 0 ]]; then
                echo "Cleaned orphaned pod volumes: $pod"
              else
                echo "ERROR: Pod cannot be deleted: ${pod_dir}" >&2
              fi
            else
              [[ $DEBUG == 1 ]] && echo "DEBUG: $pod doesn't exist in directory (${pod_dir})"
            fi
          else
            [[ $DEBUG == 1 ]] && echo "DEBUG: no orphan pod to delete"
          fi 
          sleep 3 
          echo 
        done
````

## 资料总结
&emsp;&emsp;When a Pod which has used PVC is deleted from kubuernetes cluster we can get a error message repeatedly(Orphaned pod "1f415e-...-4b38"found, but volume paths are still present on disk) from kubelet log.
this error message is caused by kubelet because it cannot clean the parent path of targetPath when Pod is deleted from cluster
(targetPath like: /var/lib/kubelet/pods/9cda187a-7fb3-11ea-80b3-246e968d4b38/volumes/kubernetes.io-csi/pvc-9ae9405c-7fb3-11ea-80b3-246e968d4b38/mount). If I remove the parent path of targetPath(/var/lib/kubelet/pods/9cda187a-7fb3-11ea-80b3-246e968d4b38/volumes/kubernetes.io~csi/pvc-9ae9405c-7fb3-11ea-80b3-246e968d4b38), kubelet stop output this error message. CSI plugin(like: ChubaoFS CSI) can remove the parent path of targetPath in NodeUnpublishVolume method.
I think that it is a better solution that fix this issue in kubelet. And other CSI Plugin can aviod this issue first using this solution before fixed kubelet.
