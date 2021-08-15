---
title: 初学 Kubernetes
tags: kubernetes
categories:
  - coding
  - kubernetes 学习笔记
date: 2021-08-15 20:48:54
---


个人kubernetes初学笔记, 跟着 <https://kubernetes.io/docs/tutorials/> 走一遍.

## 使用 `minikube` 单机部署抢先体验

### 1. 安装

本人使用的是虚拟机上的 ubuntu 21.04, 安装方法参照 `minikube` 官网, `kubectl` 官网, 过程略过.

安装完成后, 执行 `minikube start` 安装单机 kubernetes 集群, 装完后执行 `minikube dashboard` 运行监控页面, 但还需要打开代理才能让外网访问.

打开另一个 bash 窗口,

```bash
$ kubectl proxy --port=8080 --address=0.0.0.0 --accept-hosts="^.*"
Starting to serve on [::]:8080
```

复制 dashboard 的 url, 把 ip 地址改成 192.168.23.150:8080, 就能在宿主机上访问了.

例如:

http://192.168.23.150:8080/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy

![屏幕截图](/asset/初学-Kubernetes/185416.png)

### 2. 使用 kubectl 命令

#### 创建一个 deployment:

    kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1


国内可能会下载镜像失败, 可以考虑用阿里云的镜像.

查看 deploy:

    $ kubectl get deployments

结果类似

    NAME                READY   UP-TO-DATE   AVAILABLE   AGE
    kubernetes-bootcamp   1/1     1            1           1m

#### 创建 service:


    kubectl expose deployment/kubernetes-bootcamp --type="LoadBalancer" --port 8080


`--type` 有 4 种:
- ClusterIP: 只有集群内部可以访问
- NodePort: 外部可以通过 Node ip 和暴露的 port 访问
- LoadBalancer: 外部可以通过一个负载平衡服务分配的 ip 访问
- ExternalName: 映射到 `externalName` 字段 (e.g. `foo.bar.example.com`), 返回 `CNAME` 记录, 不需要 proxy. 需要 `kube-dns` v1.7+, 或 CoreDNS v0.0.8+

依次是包含关系, LoadBalancer > NodePort > ClusterIP,


#### 查看 service

    kubectl get service [option: name] [option: -l label] [option: -n namespace]

#### 查看 pod:

    kubectl get pods [option]

一般一个 pod 对应一个容器 app, 除非一组 app 有非常紧密的联系, 资源和 log 都共用, 才会将一组容器 app 定义成一个 pod.
 
#### 查看 nodes:

    kubectl get nodes [option]

#### kubectl get [type] - 列出资源

#### kubectl describe [resource] - 查看资源信息

#### kubectl logs [pod real name] - 查看 pod 的容器的 logs, 类似 docker logs

#### kubectl exec [pod real name] - 在 pod 的容器执行命令, 类似 docker exec

#### 命名简化

    deployments -> deploy
    service -> svc
    namespace -> n
    label -> l
    output -> o

#### 横向伸缩 apps

可以通过命令随时伸缩 pod, kubernetes 会通过指定好的数量指定一个 desire state 的目标, 会不断地监控 pod 的数量和状态, 努力的保证满足 desire state.

Kubernetes 通过监控资源的使用情况, 自动伸缩, 以后会学习到.

 查看 replicaSets:
    
    kubectl get rs

执行命令增加 replicas, 从 1 个增加到 4 个:

    kubectl scale deployments/kubernetes-bootcamp --replicas=4

通过命令监控伸缩的状态:

    kubectl get pods -o wide --watch

通过 describe 查看事件记录, 在底部能看到 replicas 的变化记录:

    kubectl describe deployments/kubernetes-bootcamp

通过几次的 loadBalance ip 访问 app, 增加的 app 确实有运行.

执行命令减少 replicas, 从 4 个减少到 2 个:

    kubectl scale deployments/kubernetes-bootcamp --replicas=2

同样可以通过 `--watch` 监控 pod 伸缩状态:

    kubectl get pods -o wide --watch

#### 更新 app

使用 Rolling updates 实现 0 停机时间热更新. 更新的 pod 会自动调度到有资源的 node.

在更新中, 默认最大不可用 pod 数量是 1, 默认同时最大更新数量是1, 可以通过设置 pod 数量百分比调整.

而且更新都是有版本控制的, 随时可以回滚上上一个版本.

执行命令更新 image:

    kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

通过 get 监控:

    kubectl get pods

查看更新是否已完成:

    kubectl rollout status deployments/kubernetes-bootcamp

继续更新到v10:

    kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10

通过 `kubectl get pods` 发现有一部分卡主了, 状态是 `ImagePullBackOff`.

查看更详细的信息:

    kubectl describe pods

在一些 pod 的 event 里面看到, pull v10 镜像的时候出错, 提示 `not found`.

回滚到上个版本, 也就是 v2:

    kubectl rollout undo deployments/kubernetes-bootcamp

再执行 `kubectl get pods`, 看到所有 pod 又正常运行了.

再次执行 `kubectl describe pods`, 通过 event 的 age 列, 发现 4 个里面有 1 个是新的, 符合预期.
