# [kubernetes入门]快速了解和上手容器编排工具k8s-学习笔记

> 视频链接 [【kubernetes入门】快速了解和上手容器编排工具k8s](https://www.bilibili.com/video/av61990770) https://www.bilibili.com/video/av61990770

## 概述

### 服务端

​	编排的容器不限于docker，只不过实际一般编排docker。master管理多个node节点，其中master主要有4个进程。

1. kube-apiserver管理node节点，同时也监听用户的指令（1-kubectl指令；2-RestAPI接口；3-WebUI）进行操作。
2. ETCD，它是一个数据库，用来存储一些元数据信息，比如各个节点的状态
3. controller manager，各种资源自动化的控制中心
4. scheduler，调度接口的主要实施者

整个流程大致就是，用户通过3种形式的指令下发到kube-apiserver，然后kube-apiserver起到传达的作用，需要controller manager和scheduler进行协调调度，同时ETCD提供元数据信息的支持，最后生成一个调度的指令传给apiserver，apiserver最终再把指令下发给管理的node节点，node再进行具体的容器的创建、销毁、扩张等操作，node操作完成后会把状态实时更新到apiserver，apiserver再把状态信息存储到ETCD中。

### 客户端

客户端node主要有3个重要的进程：

1. kubelet：和apiserver直接进行通信，并且可以直接调度容器。
2. kube-proxy：用来创建虚拟网卡
3. docker：实际的程序啥的在docker容器中运行

## 整合概述

​	前面提到k8s整合的容器不限于docker，所以定义了调度的最小单位是pod，一般是由一个docker和一个叫pause的东西（pause其实也是一个docker容器）组成。也就是不直接调度docker容器，而是通过更轻量的pause容器来管理调度docker容器，两者一起创建共同形成最小调度单位pod。

​	*ps：pod其实可以有多个docker容器，但是大多数时候都是一个应用容器+一个pause*

通过google搜索kubernetes online，进入第一个[Katacoda](https://www.katacoda.com/courses/kubernetes/playground)进行在线模板操作。

```shell
## 通过kubectl进行查询,查看集群运行的位置和版本
kubectl cluster-info

## 获取节点资源信息
kubectl get pod # 显示No resources found. 因为没有可用的节点资源

```

## deployment

### 概述

​	维持pod数量。比如过去原本10个机器挂了2台，需要即时打开挂的2台避免承受不了高并发访问。（过去可以通过写脚本实时监听机器数量，要是挂了就自动通知运维人员去打开，但是效率低，不够智能）

```shell
## 创建deployment
kubectl run d1 --image httpd:alpine --port 80 # kubectl run 指定名称 --image 镜像 --port  端口

## 查看启动的资源
kubectl get deployments

## 在node节点中验证httpd是否运行
docker ps |grep httpd

## 修改副本的数量
kubectl edit deployments d1 #编辑该yaml文件里面spec: replicas: 1为2，修改副本数量为2，保存后立即生效

## 查看副本数量是否改变
kubectl get deployments # 发现READY由1/1变成2/2

## 回到下方node节点，验证是否有2个docker容器运行
docker ps |grep httpd # 发现有两个容器在运行

## 强行停止第一个容器的运行
docker stop 第一个容器的ID

## 回到master查看运行的deployments，发现还是2个
kubectl get deployments

## 到node节点再次查看运行的容器数量，发现还是2个，但是第一个容器的ID和上次的不一样，明显是新建的
docker ps |grep httpd

## 也就是说我们结束掉一个pod时，deployment监听到pod数量不为设置的2，会马上再启动一个pod
```

## service

### 概述	

​	由于deployment只做到维持pod的数量，但是实际生产需要提供一个对外暴露的接口，做到负载均衡，所以引申出来service。

​	一个deployment一般会创建多个pod，而service将多个pod抽象成一个服务。

下面举个例子，有2个node节点，一个deployment分配2个pod到node1、1个pod到node2；另一个deployment分配1个pod到node1、2个pod到node2。这时候前面提到的node中的kube-proxy会在这些node上面抽象出一层虚拟交换机，给第一个deployment分配的3个pod共用IP1，而第二个deployment分配的3个pod共用IP2。外界的请求到了这个虚拟交换机的指定IP和端口后会通过负载均衡的算法均匀分配到这3个deployment管理的pod服务实例，另一个deployment同理。

```shell
## 指定deployment的d1服务向外暴露80端口，并且指定运行为NodePort
kubectl expose deployment d1 --target-port 80 --type NodePort

## 查看暴露给外界的service，可以把service简写成svc
kubectl get svc # 可以看到下面还有一个默认启动的服务，不用管，而我们启动的服务被分配了一个虚拟的IP，CLUSTER-IP

## 可以通过这个分配的虚拟IP直接访问到内部的资源
curl 10.100.126.201 # 每个人启动服务后分配的虚拟IP可能不同
```

​	这样service就帮我们解决了多容器之间做负载均衡和对外统一的接口映射的过程。

​	接下去讨论的就是不同服务之间访问的问题，就像前面docker-compose那样，不可能自己记住IP去配置。k8s也可通过deployment的名字去互相访问，而不直接手动配置IP。

```shell
## 创建第二个deployment
kubectl run d2 --image nginx:alpine --port 80

## 暴露第二个服务
kubectl expose deployment d2 --target-port=80 --type=NodePort 

## 服务1：d1 httpd镜像；服务2：d2 nginx镜像
## 查看是否开启了两个服务,查看两者被分配的虚拟IP
kubectl get svc

## 在master节点中通过sh进入进入d2的容器(参数是node查找启动的nginx对应的名称中间部分)
kubectl exec -it d2-58759f8c6-gsmlm sh
#这里查询nginx容器的名称对应为k8s_d2_d2-58759f8c6-gsmlm_default_d43d63a8-4ce3-11ea-8095-0242ac110008_0

## 首先进入容器后，安装curl工具
apk add curl

## 因为当前在d2容器中，这里直接curl d1
curl d1 # 发现能获取到httpd服务的默认信息，查看/etc/hosts发现有自动配置了一项IP路由
# 10.40.0.2       d2-58759f8c6-gsmlm
# 因为自动配置好了dns，所以curl d1 相当于curl 服务d1的虚拟ip
```

​	DNS解决了服务之间的调度问题，但是面临全新的问题，服务内部抽象出来的虚拟IP，外界用户无法知道该虚拟IP，只能知道master机器的公网IP。这里需要将虚拟IP映射到外面，让用户通过代理最终访问到内部的虚拟IP，这就是我们要提到的ingress。

## ingress

### 概述

​	ingress其实是http访问的，根据域名解析到svc，而svc本身又对应到自己的虚拟IP，这个虚拟IP再往下又集成负载均衡。

```shell
## 退出刚才的docker容器，尝试直接curl d1和d2发现无法访问到
exit
curl d1 # curl: (6) Could not resolve host: d1
curl d2 # curl: (6) Could not resolve host: d2

## 之前d1和d2之前能互相访问是通过了虚拟出来的DNS，但是外部crul不能直接访问该虚拟出来的DNS
## https://github.com/sunwu51/notebook/blob/master/19.07/ingress-deployment.yml

## ifconfig，查看本机IP
ifconfig # 172.17.0.8

## 新建一个文本文件，粘贴上面网址复制的内容，修改最后一行IP为本机IP，重新复制
vim ing-dep.yml # 粘贴并保存

## 
kubectl apply -f ing-dep.yml

## 创建另一个配置文件
## https://github.com/sunwu51/notebook/tree/master/19.07
## 修改serviceName为d1,host随便改，比如改成a.b.c
vim ing-conf.yml

## 
kubectl apply -f inf-conf.yml

## 以XX host访问本地ip curl -H "Host: xxx" ip
curl -H "Host: a.b.c" 172.17.0.8 # 获取到httpd默认的服务信息

## 在原本的host配置多加一条
vim ing-conf.yml # 复制下面的内容，修改host为a.b.d，serviceName改为h2

## 使应用生效
kubectl apply -f inf-conf.yml

## 访问到httpd服务
curl -H "Host: a.b.c" 172.17.0.8 

## 访问到nginx服务
curl -H "Host: a.b.d" 172.17.0.8 
```

通过配置本地的域名，用户可以通过不同的域名访问到不同的service，解决了master无法访问到service的问题，这样用户也可以调用到内部service的服务了

## k8s网络

> [K8S 容器之间通讯方式](https://www.jianshu.com/p/b4eabf55533d)
>
> [k8s通过service访问pod（五）](https://www.cnblogs.com/it-peng/p/11393779.html)
>
> [[k8s重要概念及部署k8s集群（一）](https://www.cnblogs.com/it-peng/p/11393762.html)]
>
> [在Play with Kubernetes平台上以测试驱动的方式部署Istio](https://www.jianshu.com/p/7f7919f598ca)
>
> [[k8s\]kubeadm k8s免费实验平台labs.play-with-k8s.com,k8s在线测试](https://www.cnblogs.com/iiiiher/p/8203529.html)

## k8s安装

> [在 MacOS 中使用 multipass 安装 microk8s 环境](https://www.cnblogs.com/gaochundong/p/install-microk8s-on-macos-using-multipass.html)