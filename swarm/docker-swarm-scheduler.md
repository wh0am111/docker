---
title: Docker Swarm服务调度分析
date: 2018-10-20 16:37:53
tags: 
 - Docker Swarm
 - Service
category: 技术 
---


本文主要介绍docker swarm 对于容器应用的调度管理分析。Docker swarm 1.12.0  是一个非常重要的版本，1.12.0之前的版本 docker swarm 是一个独立的项目，需要额外下载swarm镜像进行安装。1.12.0  之后的版本，swarm功能直接继承到docker engine 当中。直接使用docker CLI 可创建一个swarm 集群、可在swarm 集群中部署应用程序，同时具备管理能力。 

关于swarm的详细介绍，可参考官方文档：https://docs.docker.com/engine/swarm/

# 1.Docker Swarm 集群架构

Swarm 集群中，有两种类型的节点：Manager和Worker。

要部署服务到swarm，需要将服务定义提交给Manager节点。Manager节点将称为work的工作单元分派 给Worker节点。

![](.\img\swarm-node.png)

<!-- more -->

## 1.1 Manager Node

Manager节点处理集群管理任务：

- 维护集群状态
- 调度服务
- 为 Swarm 提供外部可调用的 API 接口

Manager 节点需要时刻维护和保存当前 Swarm 集群中各个节点的一致性状态，这里主要是指各个 Tasks 的执行的状态和其它节点的状态。因为 Swarm 集群是一个典型的分布式集群，在保证一致性上，Manager 节点采用 [Raft](https://raft.github.io/raft.pdf) 协议来保证分布式场景下的数据一致性。

通常为了保证 Manager 节点的高可用，Docker 建议采用奇数个 Manager 节点，这样的话，你可以在 Manager 失败的时候不用关机维护，Docker 官方给出如下的建议：

- 3 个 Manager 节点最多可以同时容忍 1 个 Manager 节点失效的情况下保证高可用；
- 5 个 Manager 节点最多可以同时容忍 2 个 Manager 节点失效的情况下保证高可用；
- N 个 Manager 节点最多可以同时容忍 (N−1)/2个 Manager 节点失效的情况下保证高可用；
- 最多最多的情况下，使用 7 个 Manager 节点就够了，否则反而会降低集群的性能了。

> **重要说明**：添加更多管理器并不意味着可扩展性更高或性能更高。一般而言，情况正好相反。

| 群体大小 | 多数 | 容错  |
| -------- | ---- | ----- |
| 1        | 1    | 0     |
| 2        | 2    | 0     |
| **3**    | 2    | **1** |
| 4        | 3    | 1     |
| **5**    | 3    | **2** |
| 6        | 4    | 2     |
| **7**    | 4    | **3** |
| 8        | 5    | 3     |

## 1.2 Worker Node 

Worker节点接收并执行从Manager节点分派的任务。默认情况下，Manager节点还将服务作为Worker节点运行，但也可以将它们配置为仅运行Manage任务。 可将Manager可用性设置为`Drain` ，这样服务就不会被分配到Manager节点上运行了。

Worker节点不参与Raft分布状态，做出调度决策或提供群模式HTTP API。

通过 docker node promote 命令将一个 Worker 节点提升为 Manager 节点。通常情况下，该命令使用在维护的过程中，需要将 Manager 节点占时下线进行维护操作。同样可以使用 docker node demote 将某个 manager 节点降级为 worker 节点。 

## 1.3 小结  

综上可知， Swarm 集群的管理工作是由manager节点实现。如上图所示，manager节点实现的功能主要包括：node discovery，scheduler,cluster管理等。同时，为了保证Manager 节点的高可用，Manager 节点需要时刻维护和保存当前 Swarm 集群中各个节点的一致性状态。在保证一致性上，Manager 节点采用 [Raft](https://raft.github.io/raft.pdf) 协议来保证分布式场景下的数据一致性；

Docker Swarm内置了Raft一致性算法，可以保证分布式系统的数据保持一致性同步。Etcd, Consul等高可用键值存储系统也是采用了这种算法。这个算法的作用简单点说就是随时保证集群中有一个Leader，由Leader接收数据更新，再同步到其他各个Follower节点。在Swarm中的作用表现为当一个Leader 节点 down掉时，系统会立即选取出另一个Leader节点，由于这个节点同步了之前节点的所有数据，所以可以无缝地管理集群。

Raft的详细解释可以参考[《The Secret Lives of Data--Raft: Understandable Distributed Consensus》](http://thesecretlivesofdata.com/raft/)。

为保证Magnger高可用，一般部署3-7个Manager 节点。而在云桌面的场景下，可能一个集群就只有两台服务器。显然此时是无法满足Manager 3个节点的高可用要求。因此我们面对的挑战就是如何修改swarm Manager Raft相关部分代码，使其在只有2个节点的情况下，也可以保证一个节点宕机的高可用。

# 2.Docker Swarm 网络架构

## 2.1 网络概述

我们知道，Docker 的几种网络方案：none、host、bridge 和 joined 容器，它们解决了单个 Docker Host 内容器通信的问题。那么跨主机容器间通信的方案又有哪些呢？

跨主机网络方案包括：

1. docker 原生的 overlay 和 macvlan。
2. 第三方方案：常用的包括 flannel、weave 和 calico。

Docker 网络是一个非常活跃的技术领域，不断有新的方案开发出来，那么要问个非常重要的问题了： 如此众多的方案是如何与 docker 集成在一起的？答案是：libnetwork 以及 CNM。

libnetwork 是 docker 容器网络库，最核心的内容是其定义的 Container Network Model (CNM)，这个模型对容器网络进行了抽象，由以下三类组件组成：

1. Sandbox：Sandbox 是容器的网络栈，包含容器的 interface、路由表和 DNS 设置。 Linux Network Namespace 是 Sandbox 的标准实现。Sandbox 可以包含来自不同 Network 的 Endpoint。
2. Endpoint：Endpoint 的作用是将 Sandbox 接入 Network。Endpoint 的典型实现是 veth pair，后面我们会举例。一个 Endpoint 只能属于一个网络，也只能属于一个 Sandbox。
3. Network：Network 包含一组 Endpoint，同一 Network 的 Endpoint 可以直接通信。Network 的实现可以是 Linux Bridge、VLAN 等。

如图所示两个容器，一个容器一个 Sandbox，每个 Sandbox 都有一个 Endpoint 连接到 Network 1，第二个 Sandbox 还有一个 Endpoint 将其接入 Network 2.

libnetwork CNM 定义了 docker 容器的网络模型，按照该模型开发出的 driver 就能与 docker daemon 协同工作，实现容器网络。docker 原生的 driver 包括 none、bridge、overlay 和 macvlan，第三方 driver 包括 flannel、weave、calico 等。

## 2.2 Overlay网络

Docker Swarm 内置的跨主机容器通信方案是overlay网络，这是一个基于VxLAN 协议的网络实现。VxLAN 可将二层数据封装到 UDP 进行传输，VxLAN 提供与 VLAN 相同的以太网二层服务，但是拥有更强的扩展性和灵活性。 overlay 通过虚拟出一个子网，让处于不同主机的容器能透明地使用这个子网。所以跨主机的容器通信就变成了在同一个子网下的容器通信，看上去就像是同一主机下的bridge网络通信。

![1534939506173](.\img\overlay-vxlan.png)

根据vxlan的作用知道，它是要在三层网络中虚拟出二层网络，即跨网段建立虚拟子网。简单的理解就是把发送到虚拟子网地址`10.0.0.3`的报文封装为发送到真实IP`192.168.1.3`的报文。这必然会有更大的数据开销，但却简化了集群的网络连接，让分布在不同主机的容器好像都在同一个主机上一样 。

`overlay`网络会创建多个Docker主机之间的分布式网络。该网络位于（覆盖）特定于主机的网络之上，允许连接到它的容器（包括群集服务容器）安全地进行通信。Docker透明地处理每个数据包与正确的Docker守护程序主机和正确的目标容器的路由。

初始化swarm或将Docker主机加入现有swarm时，会在该Docker主机上创建两个新网络：

- ingress overlay 网络，处理与swarm集群服务相关的控制和数据流量。创建群组服务并且不将其连接到用户定义的覆盖网络时，服务将默认连接到ingress overlay网络。集群中只能有一个ingress overlay 网络。
- docker_gwbridge 桥接网络，它将各个Docker守护程序连接到参与该群集的其他Docker守护进程。同时该docker_gwbridge 网络将为主机上的容器提供访问外网的能力。

服务或容器一次可以连接到多个网络。服务或容器只能通过它们各自连接的网络进行通信。

### 2.2.1 创建overlay网络

> **先决条件**：
>
> - 使用overlay 网络的Docker守护程序的防火墙规则
>
>   您需要以下端口打开来往于overlay 网络上的每个Docker主机的流量：
>
>   - 用于集群管理通信的TCP端口2377
>   - TCP和UDP端口7946用于节点之间的通信
>   - UDP端口4789用于覆盖网络流量
>
> - 在创建overlay 网络之前，您需要将Docker守护程序初始化为swarm管理器，`docker swarm init`或者使用它将其连接到现有的swarm `docker swarm join`。这些中的任何一个都会创建默认ingress overlay 网络，默认情况下 由群服务使用。即使您从未计划使用群组服务，也需要执行此操作。之后，您可以创建其他用户定义的overlay 网络。

要创建用于swarm服务的覆盖网络，请使用如下命令：

```
$ docker network create -d overlay my-overlay
```

要创建可由群集服务或独立容器用于与在其他Docker守护程序上运行的其他独立容器通信的覆盖网络，请添加`--attachable`标志：

```
$ docker network create -d overlay --attachable my-attachable-overlay
```

同时可以指定IP地址范围，子网，网关和其他选项。详情`docker network create --help`请见

### 2.2.2 加密overlay网络

默认情况下，使用GCM模式下的AES算法加密所有swarm群集服务管理流量 。集群中的管理器节点每隔12小时轮换用于加密数据的密钥。

要加密应用程序数据，请`--opt encrypted`在创建覆盖网络时添加。这样可以在vxlan级别启用IPSEC加密。此加密会产生不可忽视的性能损失，因此您应该在生产中使用此选项之前对其进行测试。

启用覆盖加密后，Docker会在所有节点之间创建IPSEC隧道，在这些节点上为连接到覆盖网络的服务安排任务。这些隧道还在GCM模式下使用AES算法，管理器节点每12小时自动旋转密钥。

```
docker network create --opt encrypted --driver overlay my-encrypted-network
```

### 2.2.3 暴露服务端口

对于连接到同一个swarm集群的服务来说，它们之间所有的端口都是互相暴露的。对于可在服务外部访问的端口，必须使用 -p ( 或者 --publish) 发布该端口。

| Flag value                                                   | Description                                         |
| ------------------------------------------------------------ | --------------------------------------------------- |
| `-p 8080:80` or `-p published=8080,target=80`                | 将服务上的TCP端口80映射到routing mesh上的端口8080。 |
| `-p 8080:80/udp` or `-p published=8080,target=80,protocol=udp` | 将服务上的UDP端口80映射到routing mesh上的端口8080。 |



### 2.2.4 实现原理

下面我们讨论下overlay 网络的具体实现：

docker 会为每个 overlay 网络创建一个独立的 network namespace，其中会有一个 linux bridge br0，endpoint 还是由 veth pair 实现，一端连接到容器中（即 eth0），另一端连接到 namespace 的 br0 上。

br0 除了连接所有的 endpoint，还会连接一个 vxlan 设备，用于与其他 host 建立 vxlan tunnel。容器之间的数据就是通过这个 tunnel 通信的。逻辑网络拓扑结构如图所示：

![](./img/overlay.jpg)



## 2.3 Ingress Routing Mesh

上文暴露服务端口中有讲到routing mesh，现在来详细介绍这一网络功能。

默认情况下，发布端口的swarm服务使用routing mesh来实现。当客户端连接到任何swarm节点上的已发布端口（无论它是否正在运行给定服务）时，客户端请求将被透明地重定向到正在运行该服务的worker。实际上，Docker充当集群服务的负载均衡器。使用routing mesh的服务以*虚拟IP（VIP）模式运行*。即使在每个节点上运行的服务（通过`--global`标志）也使用routing mesh。使用routing mesh时，无法保证哪个Docker节点服务客户端请求。

要在群集中使用 ingress 网络，您需要在启用群集模式之前在群集节点之间打开以下端口：

- 端口`7946`  TCP / UDP用于容器网络发现。
- 端口`4789`  UDP 用于容器 ingress 网络。

例如，以下命令将nginx容器中的端口80发布到群集中任何节点的端口8080：

```
$ docker service create \
  --name my-web \
  --publish published=8080,target=80 \
  --replicas 2 \
  nginx
```

当您在任何节点上访问端口8080时，Docker会将您的请求路由到活动容器。在群集节点本身上，端口8080实际上可能不受约束，但routing mesh 知道如何路由流量并防止发生任何端口冲突。

routing mesh在已发布的端口上侦听分配给该节点的任何IP地址。对于可外部路由的IP地址，该端口可从主机外部获得。对于所有其他IP地址，只能从主机内访问。

![](.\img\ingress-routing-mesh.png)

如果想要绕过routing mesh，可以使用*DNS循环（DNSRR）模式*启动服务，方法是将`--endpoint-mode`标志设置为`dnsrr`。这样就必须在服务前运行自己的负载均衡器。Docker主机上的服务名称的DNS查询返回运行该服务的节点的IP地址列表。配置负载均衡器以使用此列表并平衡节点之间的流量。

对于我们云桌面应用场景来说，直接利用routing mesh 功能即可，不需要额外搭建负载均衡器。

## 2.4 服务发现

Docker Swarm 原生就提供了服务发现的功能。要使用服务发现，需要相互通信的 services 必须属于同一个 overlay 网络，所以我们先得创建一个新的 overlay 网络。 直接使用 `ingress` 行不行？很遗憾，目前 `ingress` 没有提供服务发现，必须创建自己的 overlay 网络。

Docker Swarm mode下会为每个节点的Docker engine内置一个DNS server，各个节点间的DNS server通过control plane的gossip协议互相交互信息。此处DNS server用于容器间的服务发现。swarm mode会为每个 --net=自定义网络  的service分配一个DNS entry。目前必须是自定义网络，比如overaly。而bridge和routing mesh 的service，是不会分配DNS的。

那么，下面就来详细介绍服务发现的原理。

每个Docker容器都有一个DNS解析器，它将DNS查询转发到docker engine，该引擎充当DNS服务器。docker 引擎收到请求后就会在发出请求的容器所在的所有网络中，检查域名对应的是不是一个容器或者是服务，如果是，docker引擎就会从存储的key-value建值对中查找这个容器名、任务名、或者服务名对应的IP地址，并把这个IP地址或者是服务的虚拟IP地址返回给发起请求的域名解析器。  

由上可知，docker的服务发现的作用范围是网络级别，也就意味着只有在同一个网络上的容器或任务才能利用内嵌的DNS服务来相互发现，不在同一个网络里面的服务是不能解析名称的，另外，为了安全和性能只有当一个节点上有容器或任务在某个网络里面时，这个节点才会存储那个网络里面的DNS记录。

如果目的容器或服务和源容器不在同一个网络里面，Docker引擎会把这个DNS查询转发到配置的默认DNS服务  。

![](.\img\service-dns.png)

在上面的例子中，总共有两个服务myservice和client，其中myservice有两个容器，这两个服务在同一个网里面。在client里针对docker.com和myservice各执行了一个curl操作，下面时执行的流程： 

- 为了client解析docker.com和myservice，DNS查询进行初始化
- 容器内建的解析器在127.0.0.11:53拦截到这个DNS查询请求，并把请求转发到docker引擎的DNS服务
- myservice 被解析成服务对应的虚拟IP（10.0.0.3）。在接下来的内部负载均衡阶段再被解析成一个具体任务容器的IP地址。如果是容器名称这一步直接解析成容器对应的IP地址（10.0.0.4或者10.0.0.5）。
- docker.com 在docker引擎或者mynet网络上不能被解析成服务，所以这个请求被转发到外部DNS，例如配置好的默认DNS服务器（8.8.8.8）上。

Docker1.12+的服务发现和负载均衡是结合到一起的， 实现办法有两种，DNS轮询和IPVS。 配置参数 --endpoint-mode  (vip or dnsrr) ，默认使用VIP方式。DNS轮询有一些缺点，比如一些应用可能缓存DNS请求， 导致容器变化之后DNS不能实时更新；DNS生效时间也导致不能实时反映服务变化情况。VIP和IPVS原理上比较容易理解， 就是Docker为每个服务分配了一个VIP，DNS解析服务名称或者自定义的别名到这个VIP上。由于VIP本身没有容器提供服务，Docker把到VIP的请求通过IPVS技术负载到后面的容器上。

因此正常情况下，使用默认的VIP方式即可。

## 3.5 负载均衡

负载均衡分为两种：

Swarm集群内的service之间的相互访问需要做负载均衡，称为内部负载均衡（Internal LB）；

从Swarm集群外部访问服务的公开端口，也需要做负载均衡，称外部负载均衡(Exteral LB or Ingress LB)。

### 3.5.1 Internal LB

内部负载均衡就是我们在上一段提到的服务发现，集群内部通过DNS访问service时，Swarm默认通过VIP（virtual IP）、iptables、IPVS转发到某个容器。 

![](.\img\intelnal-lb.jpg)

当在docker swarm集群模式下创建一个服务时，会自动在服务所属的网络上给服务额外的分配一个虚拟IP，当解析服务名字时就会返回这个虚拟IP。对虚拟IP的请求会通过overlay网络自动的负载到这个服务所有的健康任务上。这个方式也避免了客户端的负载均衡，因为只有单独的一个IP会返回到客户端，docker会处理虚拟IP到具体任务的路由，并把请求平均的分配给所有的健康任务。  

![](.\img\internal-lb-2.png)

```
# 创建overlay网络：mynet 
$ docker network create -d overlay mynet  
a59umzkdj2r0ua7x8jxd84dhr 
# 利用mynet网络创建myservice服务，并复制两份  
$ docker service create --network mynet --name myservice --replicas 2 busybox ping localhost  
78t5r8cr0f0h6k2c3k7ih4l6f5
# 通过下面的命令查看myservice对应的虚拟IP 
$ docker service inspect myservice  
...
"VirtualIPs": [ 
	{  
     "NetworkID": "a59umzkdj2r0ua7x8jxd84dhr",  
     			"Addr": "10.0.0.3/24"  
      },  
]  	
```

注：swarm中服务还有另外一种负载均衡技术可选DNS round robin (DNS RR) （在创建服务时通过--endpoint-mode配置项指定），在DNSRR模式下，docker不再为服务创建VIP，docker DNS服务直接利用轮询的策略把服务名称直接解析成一个容器的IP地址。 

### 3.5.2 Exteral LB

Exteral LB(Ingress LB 或者 Swarm Mode Routing Mesh) 看名字就知道，这个负载均衡方式和前面提到的Ingress网络有关。Swarm网络要提供对外访问的服务就需要打开公开端口，并映射到宿主机。Exteral LB就是外部通过公开端口访问集群时做的负载均衡。

当创建或更新一个服务时，你可以利用--publish选项把一个服务暴露到外部，在docker swarm模式下发布一个端口意味着在集群中的所有节点都会监听这个端口，这时当访问一个监听了端口但是并没有对应服务运行在其上的节点会发生什么呢？接下来就该我们的路由网（routing mesh）出场了，路由网时docker1.12引入的一个新特性，它结合了IPVS和iptables创建了一个强大的集群范围的L4层负载均衡，它使所有节点接收服务暴露端口的请求成为可能。当任意节点接收到针对某个服务暴露的TCP/UDP端口的请求时，这个节点会利用预先定义过的Ingress overlay网络，把请求转发给服务对应的虚拟IP。ingress网络和其他的overlay网络一样，只是它的目的是为了转换来自客户端到集群的请求，它也是利用我们前一小节介绍过的基于VIP的负载均衡技术。 

启动服务后，你可以为应用程序创建外部DNS记录，并将其映射到任何或所有Docker swarm节点。你无需担心你的容器具体运行在那个节点上，因为有了路由网这个特性后，你的集群看起来就像是单独的一个节点一样。  

```
#在集群中创建一个复制两份的服务，并暴露在8000端口  
$ docker service create --name app --replicas 2 --network appnet --publish 8000:80 nginx  
```

![](.\img\external-routing-mesh.png)

上面这个图表明了路由网是怎么工作的： 

- 服务（app）拥有两份复制，并把端口映射到外部端口的8000
- 路由网在集群中的所有节点上都暴露出8000
- 外部对服务app的请求可以是任意节点，在本例子中外部的负载均衡器将请求转发到了没有app服务的主机上
- docker swarm的IPVS利用ingress overlay网路将请求重新转发到运行着app服务的节点的容器中

注：以上服务发现和负载均衡参考文档 https://success.docker.com/article/ucp-service-discovery



# 3.Docker Swarm 存储架构

从业务数据的角度看，容器可以分为两类：无状态（stateless）容器和有状态（stateful）容器。

无状态是指容器在运行过程中不需要保存数据，每次访问的结果不依赖上一次访问，比如提供静态页面的 web 服务器。有状态是指容器需要保存数据，而且数据会发生变化，访问的结果依赖之前请求的处理结果，最典型的就是数据库服务器。

简单来讲，状态（state）就是数据，如果容器需要处理并存储数据，它就是有状态的，反之则无状态。

对于有状态的容器，如何保存数据呢？

## 3.1 Storage Driver

Docker在启动容器的时候，需要创建文件系统，为rootfs提供挂载点。最底层的引导文件系统bootfs主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 rootfs包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。

Docker 模型的核心部分是有效利用分层镜像机制，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。Docker1.10引入新的可寻址存储模型，使用安全内容哈希代替随机的UUID管理镜像。同时，Docker提供了迁移工具，将已经存在的镜像迁移到新模型上。不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的可读写层，大大提高了存储的效率。其中主要的机制就是分层模型和将不同目录挂载到同一个虚拟文件系统。

Docker存储方式提供管理分层镜像和容器的可读写层的具体实现。最初Docker仅能在支持AUFS文件系统的ubuntu发行版上运行，但是由于AUFS未能加入Linux内核，为了寻求兼容性、扩展性，Docker在内部通过graphdriver机制这种可扩展的方式来实现对不同文件系统的支持。

![](.\img\storage-driver.png)

如上图所示，容器由最上面一个可写的容器层，以及若干只读的镜像层组成，容器的数据就存放在这些层中。这样的分层结构最大的特性是 Copy-on-Write：

1. 新数据会直接存放在最上面的容器层。
2. 修改现有数据会先从镜像层将数据复制到容器层，修改后的数据直接保存在容器层中，镜像层保持不变。
3. 如果多个层中有命名相同的文件，用户只能看到最上面那层中的文件。

分层结构使镜像和容器的创建、共享以及分发变得非常高效，而这些都要归功于 Docker storage driver。正是 storage driver 实现了多层数据的堆叠并为用户提供一个单一的合并之后的统一视图。

Docker 支持多种 storage driver，有 AUFS、Device Mapper、Btrfs、OverlayFS、VFS 和 ZFS。它们都能实现分层的架构，同时又有各自的特性。关于这些方案的对比可以查阅相关文档（http://dockone.io/article/1513）。对于 Docker 用户来说，具体选择使用哪个 storage driver 是一个难题，不过 Docker 官方给出了一个简单的答案： **优先使用 Linux 发行版默认的 storage driver**。 使用`docker info `查看系统默认的storage driver。

使用的storage driver是与主机上的Backing Filesystem有关的。storage driver 是专门用来存放docker容器和镜像的，Backing Filesystem指的是主机的文件系统。默认的centos7 driver 是overlay2，对应的Backing Filesystem 是ext4。这个应该是没有问题的。storage driver 和Backing Filesystem 对应关系见下表。 

![1534987414895](.\img\storage_backfing.png)

对于某些容器，直接将数据放在由 storage driver 维护的层中是很好的选择，比如那些无状态的应用。无状态意味着容器没有需要持久化的数据，随时可以从镜像直接创建。

比如 busybox，它是一个工具箱，我们启动 busybox 是为了执行诸如 wget，ping 之类的命令，不需要保存数据供以后使用，使用完直接退出，容器删除时存放在容器层中的工作数据也一起被删除，这没问题，下次再启动新容器即可。

但对于另一类应用这种方式就不合适了，它们有持久化数据的需求，容器启动时需要加载已有的数据，容器销毁时希望保留产生的新数据，也就是说，这类容器是有状态的。

## 3.2 数据持久化方案

那么对于有状态的容器，如何保存数据呢？

选项一：打包在容器里。

显然不行。除非数据不会发生变化，否则，如何在多个副本直接保持同步呢？

选项二：数据放在 Docker 主机的本地目录中，通过 volume 映射到容器里。

位于同一个主机的副本倒是能够共享这个 volume，但不同主机中的副本如何同步呢？

选项三：利用 Docker 的 volume driver，由外部 storage provider 管理和提供 volume，所有 Docker 主机 volume 将挂载到各个副本。

这是目前最佳的方案。volume 不依赖 Docker 主机和容器，生命周期由 storage provider 管理，volume 的高可用和数据有效性也全权由 provider 负责，Docker 只管使用。

我们知道Docker 单机的存储方案里面，通过 data volume 可以存储容器的状态。不过其本质是 Docker 主机本地的目录。 本地目录就存在一个隐患：如果 Docker Host 宕机了，如何恢复容器？ 一个办法就是定期备份数据，但这种方案还是会丢失从上次备份到宕机这段时间的数据。更好的方案是由专门的 storage provider 提供 volume，Docker 从 provider 那里获取 volume 并挂载到容器。这样即使 Host 挂了，也可以立刻在其他可用 Host 上启动相同镜像的容器，挂载之前使用的 volume，这样就不会有数据丢失。

还有一个容易混淆的存储方案，是通过在Docker主机上挂载共享文件系统（例如Ceph，GlusterFS，NFS）。如下图所示：

![](.\Chart_Multi-Host-Persistence-Shared-Among-Containers.png)

在运行Docker容器的每个主机上配置分布式文件系统。通过创建一致的命名约定和统一命名空间，所有正在运行的容器都可以访问底层的共享存储后端。共享文件系统存储方案利用分布式文件系统与显式存储技术相结合。由于挂载点在所有节点上都可用，因此可以利用它在容器之间创建共享挂载点。这种方案其实是属于选项二的一种变种。

尽管该方案对于特定用例而言，是Docker 存储方案的一个有价值的补充，但它具有限制容器到特定主机的可移植性的显着缺点。它也没有利用针对数据密集型工作负载优化的专用存储后端。为了解决这些限制，volume plugins 被添加到Docker中，将容器的功能扩展到各种存储后端，并且而无需强制更改应用程序设计或部署体系结构。 这就是我们下节将详细介绍的 Volume Driver。

##  3.3 Volume Driver

假设有两个 Dokcer 主机，Host1 运行了一个 MySQL 容器，为了保护数据，data volume 由 storage provider 提供，如下图所示。 

![](.\img\storage-provider-1.png)

当 Host1 发生故障，我们会在 Host2 上启动相同的 MySQL 镜像，并挂载 data volume。 

![](.\img\storage-provider-2.png)

Docker 是如何实现这个跨主机管理 data volume 方案的呢？

**答案是 volume driver。**

利用 Docker 的 volume driver，由外部 storage provider 管理和提供 volume，所有 Docker 主机 volume 将挂载到各个副本。任何一个 data volume 都是由 driver 管理的，创建 volume 时如果不特别指定，将使用 `local` 类型的 driver，即从 Docker Host 的本地目录中分配存储空间。如果要支持跨主机的 volume，则需要使用第三方 driver。

目前已经有很多可用的 driver，比如使用 Azure File Storage 的 driver，使用 GlusterFS 的 driver，完整的列表可参考 https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins 

![](.\img\Docker-Volume-Plugin-Architecture.png)

Volume drivers 让我们将应用程序与底层存储系统隔离。例如，如果原先服务使用一个 NFS驱动器，现在可以在不改变应用程序的逻辑下，直接更新服务使用不同的驱动程序，例如将数据存储在云端,。 

通过`docker volume create ` 命令可以直接创建volume，同时在参数中指定driver 类型。	下面的例子使用vieux/sshfs  这种类型的Volume drivers。首先创建一个单独的 volume，然后创建容器引用该volume。

### 3.3.1 安装driver插件 

这个示例假设有两个节点，两个节点直接可以通过ssh互相连接。在Docker 主机上，安装`vieux/sshfs` 插件。

```docker plugin install --grant-all-permissions vieux/sshfs
docker plugin install --grant-all-permissions vieux/sshfs
```

### 3.3.2 创建volume

本例中指定一个SSH密码，但如果两个主机配置了共享密钥，可以省略的密码。每个volume driver 可以有零个或多个可配置选项，每一个都指定使用 -o 标志。 

```
docker volume create --driver vieux/sshfs \
  -o sshcmd=test@node2:/home/test \
  -o password=testpassword \
  sshvolume
```

### 3.3.3 使用volume

本例中指定一个SSH密码,但如果两个主机配置了共享密钥,您可以省略的密码。每个卷司机可以有零个或多个可配置的选项。如果 **volume driver**  需要传递参数选项,你必须使用 **--mount**  挂载卷,而不是使用 -v。 

```
$ docker run -d \
  --name sshfs-container \
  --volume-driver vieux/sshfs \
  --mount src=sshvolume,target=/app,volume-opt=sshcmd=test@node2:/home/test,volume-opt=password=testpassword \
  nginx:latest
```

关于volume driver更多的介绍，可以查看官方文档：https://docs.docker.com/storage/volumes/#use-a-volume-driver

https://docs.docker.com/storage/volumes/

# 4.Docker Swarm 服务架构

我们知道，Docker 场景下，应用是以容器的形式运行并提供服务的。而在Swarm场景下，任务是一个个具体的容器，通过抽象出服务的概念，来管理具有一定关联关系的任务容器。 

为了将某一个服务封装为 Service 在Swarm 中部署执行，我们需要通过指定容器 image 以及需要在容器中执行的 Commands 来创建你的 Service，除此之外，还需要配置如下选项，

- 指定可以在 Swarm 之外可以被访问的服务端口号 port
- 指定加入某个 Overlay 网络以便 Service 与 Service 之间可以建立连接并通讯，
- 指定该 Service 所要使用的 CPU 和 内存的大小，
- 指定一个滚动更新的策略 (Rolling Update Policy)
- 指定多少个 Task 的副本 (replicas) 在 Swarm 集群中同时存在

如下所示，`docker-swarm.yml`文件是一个YAML文件，它定义了如何Docker容器在生产中应表现。

```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```

该`docker-swarm.yml`文件告诉Docker执行以下操作：

- 拉username/repo 仓库中拉取镜像。
- 将该映像的5个实例作为一个被调用的服务运行`web`，限制每个实例使用，最多10％的CPU（跨所有内核）和50MB的RAM。
- 如果一个失败，立即重启容器。
- 将主机上的端口4000映射到`web`端口80。
- 指示`web`容器通过称为负载平衡的网络共享端口80 `webnet`。（在内部，容器本身`web`在短暂的端口发布到 80端口。）
- `webnet`使用默认设置（负载平衡的覆盖网络）定义网络。

## 4.1 服务和任务

我们一般使用如下命令来部署一个服务到swarm中：

`docker stack deploy -c docker-swarm.yml myservice。`

docker-swarm.yml中定义了服务所需要达到的状态。我们不需要告诉swarm如何做，我们只需要告诉swarm 我们想要的服务状态是什么，swarm最终会帮我们达到这个状态。

swarm管理器将服务定义为服务的所需状态。swarm将群中节点上的服务调度为一个或多个副本任务。任务在群集中的节点上彼此独立地运行。

例如，想要在HTTP侦听器的三个实例之间进行负载平衡。下图显示了具有三个副本的HTTP侦听器服务。监听器的三个实例中的每一个都是集群中的任务。容器是一个孤立的过程。在swarm模式模型中，每个任务只调用一个容器。任务类似于调度程序放置容器的“槽”。容器生效后，调度程序会识别该任务处于运行状态。如果容器未通过运行状况检查或终止，则任务将终止。

![](.\img\services-diagram.png)

Services，Tasks 和 Containers 之间的关系可以用上面这张图来描述。来看这张图的逻辑，表示用户想要通过 Manager 节点部署一个有 3 个副本的 Nginx 的 Service，Manager 节点接收到用户的 Service definition 后，便开始对该 Service 进行调度，将会在当前可用的Worker（或者Manager ）节点中启动相应的 Tasks 以及相关的副本；所以可以看到，Service 实际上是 Task 的定义，而 Task 则是执行在节点上的程序。

Task 是什么呢？其实就是一个 Container，只是，在 Swarm 中，每个 Task 都有自己的名字和编号，如图，比如 nginx.1、nginx.2 和 nginx.3，这些 Container 各自运行在各自的 node 上，当然，一个 node 上可以运行多个 Container；

## 4.2 任务调度

任务是swarm集群内调度的原子单位。在创建或更新服务时，声明定义服务状态，协调器通过调度任务来实现所需的状态。例如上例的服务，该服务指示协调器始终保持三个HTTP侦听器实例的运行。协调器通过创建三个任务来响应。每个任务都是调度程序通过生成容器来填充的插槽。容器是任务的实例化。如果HTTP侦听器任务随后未通过其运行状况检查或崩溃，则协调器会创建一个新的副本任务，该任务会生成一个新容器。

任务是单向机制。它通过一系列状态单调进行：已分配，准备，运行等。如果任务失败，则协调器将删除任务及其容器，然后根据服务指定的所需状态创建新任务以替换它。

Docker swarm mode 模式的底层技术实际上就是指的是调度器( scheduler )和编排器( orchestrator )；下面这张图展示了 Swarm mode 如何从一个 Service 的创建请求并且成功将该 Service 分发到 worker 节点上执行的过程。

![](.\img\swarm-service-lifecycle.png)



- 首先，看上半部分Swarm manager 
  1. 用户通过 Docker Engine Client 使用命令 docker service create 提交 Service definition，
  2. 根据 Service definition 创建相应的 Task，
  3. 为 Task 分配 IP 地址，
     注意，这是分配运行在 Swarm 集群中 Container 的 IP 地址，该 IP 地址最佳的分配地点是在这里，因为 Manager 节点上保存得有最新最全的 Tasks 的状态信息，为了保证不与其他的 Task 分配到相同的 IP，所以在这里就将 IP 地址给初始化好；
  4. 将 Task 分发到 Node 上，可以是 Manager 节点也可以使 Worker 节点，
  5. 对 Worker 节点进行相应的初始化使得它可以执行 Task
- 接着，看下半部分Swarm  Work
  1. 首先连接 manager 的分配器( scheduler)检查该 task
  2. 验证通过以后，便开始通过 Worker 节点上的执行器( executor )执行；

注意，上述 task 的执行过程是一种单向机制，比如它会按顺序的依次经历 assigned, prepared 和 running 等执行状态，不过在某些特殊情况下，在执行过程中，某个 task 失败了( fails )，编排器( orchestrator )直接将该 task 以及它的 container 给删除掉，然后在其它节点上另外创建并执行该 task；

## 4.3 任务状态

通过运行`docker service ps <service-name>`以获取任务的状态。该 `CURRENT STATE`字段显示任务的状态以及任务的持续时间 。

```
[root@swarm01 ~]# docker service ps myweb_web
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                         PORTS
52lolsfykj4t        myweb_web.1         httpd:v2            swarm03             Running             Running 16 minutes ago
zwpufyfacagy         \_ myweb_web.1     httpd:v2            swarm03             Shutdown            Failed 16 minutes ago    "task: non-zero exit (255)"
00fhp9k1byf6        myweb_web.2         httpd:v2            swarm02             Running             Running 16 minutes ago
kgtiyw39xpcc         \_ myweb_web.2     httpd:v2            swarm02             Shutdown            Failed 17 minutes ago    "task: non-zero exit (255)"
vefk4yn0b00m        myweb_web.3         httpd:v2            swarm01             Running             Running 16 minutes ago
pnj4z0j281ng         \_ myweb_web.3     httpd:v2            swarm01             Shutdown            Failed 17 minutes ago    "task: non-zero exit (255)"
```

任务具有如下几种状态：

| 任务状态    | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `NEW`       | 任务已初始化。                                               |
| `PENDING`   | 分配了任务的资源。                                           |
| `ASSIGNED`  | Docker将任务分配给节点。                                     |
| `ACCEPTED`  | 该任务被工作节点接受。如果工作节点拒绝该任务，则状态将更改为`REJECTED`。 |
| `PREPARING` | Docker正在准备任务。                                         |
| `STARTING`  | Docker正在开始这项任务。                                     |
| `RUNNING`   | 任务正在执行。                                               |
| `COMPLETE`  | 任务退出时没有错误代码。                                     |
| `FAILED`    | 任务退出并显示错误代码。                                     |
| `SHUTDOWN`  | Docker请求关闭任务。                                         |
| `REJECTED`  | 工作节点拒绝了该任务。                                       |
| `ORPHANED`  | 该节点停机时间过长。                                         |
| `REMOVE`    | 该任务不是终端，但关联的服务已被删除或缩小。                 |

如果在某个 service 的调度过程中，发现当前没有可用的 node 资源可以执行该 service，这个时候，该 service 的状态将会保持为 pending 的状态；下面，我们来看一些例子可能使得 service 维持在 pending 状态，

- 如果当所有的节点都被停止或者进入了 drained 状态，这个时候，你试图创建一个 service，该 service 将会一直保持 pending 状态直到当前某个节点可用为止；不过要注意的是，第一个恢复的 node 将会得到所有的 task 调度请求，接收并执行，因此，这种情况在 production 环境上要尽量避免；
- 你可以为你的 service 设置执行所需的内存大小，在调度的过程当中，发现没有一个 node 能够满足你的内存请求，那么你当前的 service 将会一直处于 pending 状态直到有满足需求的 node 出现；
- 可以对服务施加放置约束，并且可能无法在给定时间遵守约束。 

上面的例子都表明一个事实，就是你期望的执行条件与 Swarm 现有的可用资源的情况不匹配，不吻合，因此，作为 Swarm 的管理者，应该考虑对 Swarm 集群整体进行扩容；

## 4.4 服务副本与全局服务

Docker Swarm 有两种类型的服务部署模式：复制和全局。

Swarm通过`--mode`选项设置服务类型，提供了两种模式：一种是replicated，我们可以指定服务Task的个数（也就是需要创建几个冗余副本），这也是Swarm默认使用的服务类型；另一种是global，这样会在Swarm集群的每个Node上都创建一个服务。如下图所示（出自Docker官网），是一个包含replicated和global模式的Swarm集群： 

![](.\img\docker-swarm-replicated-vs-global.png)



上图中，黄色表示的replicated模式下的Service Replicas，灰色表示global模式下Service的分布。 

在Swarm mode下使用Docker，可以实现部署运行服务、服务扩容缩容、删除服务、滚动更新等功能。

## 4.5 调度策略

Swarm提供了几种不同的方法来控制服务任务可以在哪些节点上运行。

- 可以指定服务是需要运行特定数量的副本还是应该在每个工作节点上全局运行。请参阅2.4服务副本与全局服务。

- 您可以配置服务的 CPU或内存要求，该服务仅在满足这些要求的节点上运行。

- 通过设置约束标签 --constraint ，您可以将服务配置为仅在具有特定（任意）元数据集的节点上运行，并且如果不存在适当的节点，则会导致部署失败。例如，您可以指定您的服务只应在任意标签`pci_compliant`设置为的节点上运行 `true`。

- 通过设置首选项 --preferences，您可以将具有一系列值的任意标签应用于每个节点，并使用算法将服务的任务分布到这些节点上。目前，唯一支持的算法是`spread`，该算法将尝试均匀放置它们。例如，如果`rack`使用值为1-10 的标签标记每个节点，然后指定键入的放置首选项`rack`，服务任务将尽可能均匀地放置在具有标签的所有节点上。如果具有`rack`标签的节点个数，大于任务个数，那么则会根据rack 值的大小排序，然后优先选择排名靠前的节点。

  与约束不同，放置首选项是尽力而为，如果没有节点可以满足首选项，则服务不会失败。如果为服务指定放置首选项，则当集群管理器决定哪些节点应运行服务任务时，与该首选项匹配的节点的排名会更高。其他因素，例如服务的高可用性，也是计划节点运行服务任务的因素。例如，如果您有N个节点带有机架标签（然后是其他一些节点），并且您的服务配置为运行N + 1个副本，那么+1将在一个尚未拥有该服务的节点上进行调度，如果无论该节点是否具有`rack`标签。



## 4.6 服务管理

docker swarm 还可以使用 stack 这一模型，用来部署管理和关联不同的服务。stack 功能比service 还要强大，因为它具备多服务编排部署的能力。**stack file** 是一种 yaml 格式的文件，类似于 docker-compose.yml 文件，它定义了一个或多个服务，并定义了服务的环境变量、部署标签、容器数量以及相关的环境特定配置等。 

以下实验仅仅创建利用stack 创建单一服务。

### 4.6.1 创建服务

服务配置文件docker-swarm.yml ，内容如下

```
version: '3'
services:
  web:
    image: "httpd:v1"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
      restart_policy:
        condition: on-failure
    ports:
     - "7080:80"
    volumes:
     - /var/www/html:/usr/local/apache2/htdocs
```

image: "httpd" ：httpd是docker hub 下载的镜像。提供简单的httpd服务，此处用来做服务都高可用验证。

replicas: 3 表示该服务副本有3个，可对服务副本进行扩缩容，后续会讲到。

/var/www/html:/usr/local/apache2/htdocs ：表示httpd 首页的内容。各个swarm节点上展示的内容分别是其主机名信息。

在docker-swarm.yml 同级目录下，执行命令

```
[root@swarm01 swarm]# docker stack deploy -c docker-swarm.yml myservice
Creating network myservice_default
Creating service myservice_web
```

### 4.6.2 查看服务

查看服务创建是否成功。

```
[root@swarm01 swarm]# docker stack ls
NAME                SERVICES
myservice           1
[root@swarm01 swarm]# docker stack services myservice
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
kom8x49f7qv2        myservice_web       replicated          3/3                 httpd:latest        *:7080->80/tcp
[root@swarm01 swarm]# docker stack ps myservice
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
sybn1dur1wmw        myservice_web.1     httpd:latest        swarm03             Running             Running 45 seconds ago
mlcb5r9vzv6z        myservice_web.2     httpd:latest        swarm03             Running             Running 45 seconds ago
fw21lcu3zp83        myservice_web.3     httpd:latest        swarm01             Running             Running 45 seconds ago
```

可以看到 swarm01 上面运行了1个容器，swarm03上面运行了2个，而swarm02上面一个容器都没有，这是咋回事呢？哦，原来我们在创建集群过程中，对swarm02 做了状态变更的操作，将其设置为不运行任务。` docker node update  --availability drain  swarm02`. 查看集群状态。

```
[root@swarm01 swarm]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
epoic1y0vv830vnwbc6nnacjc *   swarm01             Ready               Active              Leader              18.03.1-ce
nsqqv181e0r7mu77azixtqhzl     swarm02             Ready               Drain               Reachable           18.03.1-ce
mkkxaeqergz8xw21bbkd47q1c     swarm03             Ready               Active              Reachable           18.03.1-ce
```

此时打开浏览器，输入HTTP URL。任意swarm集群节点都可以访问httpd服务。

![](.\img\swarm-web.png)

需要说明的是，由于3个节点的/var/www/html 路径未做共享存储。因此swarm会随机显示一个swarm节点的网页内容。

### 4.6.3 服务扩缩容

修改docker-swarm.yml 配置字段 `replicas: 5` ，将容器个数提升到5个。由于实验环境资源有限，因此这里将swarm02状态变更为可以状态，用以分配容器。

` docker node update  --availability active swarm02`.

重新执行部署命令`docker stack deploy -c docker-swarm.yml myservice`

也可以通过命令直接对服务进行操作，但是不推荐。 `docker service scale 服务ID=服务个数`

```
[root@swarm01 swarm]# docker stack ps myservice
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                    ERROR               PORTS
unwepav5z9nz        myservice_web.1     httpd:v1            swarm02             Running             Running less than a second ago
3j5o1tktcg1l        myservice_web.2     httpd:v1            swarm01             Running             Preparing 3 seconds ago
wl1unt6lynft        myservice_web.3     httpd:v1            swarm03             Running             Running less than a second ago
[root@swarm01 swarm]# vi docker-swarm.yml
[root@swarm01 swarm]# docker stack deploy -c docker-swarm.yml myservice
Updating service myservice_web (id: 85quzwi5k0btkn5wlht4jbco9)
image httpd:v1 could not be accessed on a registry to record
its digest. Each node will access httpd:v1 independently,
possibly leading to different nodes running different
versions of the image.

[root@swarm01 swarm]# docker stack ps myservice
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                    ERROR               PORTS
unwepav5z9nz        myservice_web.1     httpd:v1            swarm02             Running             Running 24 seconds ago
3j5o1tktcg1l        myservice_web.2     httpd:v1            swarm01             Running             Running 31 seconds ago
wl1unt6lynft        myservice_web.3     httpd:v1            swarm03             Running             Running 24 seconds ago
ujedjxbuffxp        myservice_web.4     httpd:v1            swarm01             Running             Running less than a second ago
ml9qvjh2add7        myservice_web.5     httpd:v1            swarm02             Running             Running less than a second ago
```

同样，对于服务缩容，也仅需要修改`replicas:` 配置即可。这里不再演示。

### 4.6.4 删除服务

例如，删除myredis应用服务，执行docker service rm myredis，则应用服务myredis的全部副本都会被删除。 

本例使用的是stack ，删除stack ，即可删除其下的服务。没错，docker语法非常类似,`docker stack rm myservice`。

需要注意的是，对swarm上面服务的操作，都必须在manager节点上运行。

### 4.6.5 服务升降级

服务升降级，对应到docker中，则利用了容器镜像的更新，升级很好理解，直接把tag 标签版本往上提。那么降级，其实也可以理解为另外一种“升级“，只是镜像内容是上一个版本的而已。对于镜像标签的命令应该遵循一套严格的规范，这里不再赘述。

先介绍利用service 命令行工具的用法。

服务的滚动更新，这里我参考官网文档的例子说明。在Manager Node上执行如下命令：

```
docker service create --replicas 3  --name redis --update-delay 10s redis:3.0.6
```

上面通过指定 --update-delay 选项，表示需要进行更新的服务，每次成功部署一个，延迟10秒钟，然后再更新下一个服务。如果某个服务更新失败，则Swarm的调度器就会暂停本次服务的部署更新。

另外，也可以更新已经部署的服务所在容器中使用的Image的版本，例如执行如下命令：

将Redis服务对应的Image版本有3.0.6更新为3.0.7，同样，如果更新失败，则暂停本次更新。

那么，stack file 如何定义滚动更新呢？

```
      update_config:
        parallelism: 2
        delay: 10s
```

修改docker-swarm.yml 如下

```
version: '3'
services:
  web:
    image: "httpd:v2"
    deploy:
      replicas: 3
      update_config:
        parallelism: 2
        delay: 30s
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
      restart_policy:
        condition: on-failure
    ports:
     - "7080:80"
    volumes:
     - /var/www/html:/usr/local/apache2/htdocs
```

parallelism 表示同时升级的个数，delay 则表示间隔多长时间升级。

同时将httpd 由v1 升级到v2，这里我们使用同一个镜像，只是tag 修改了一下。查看服务状态

```
[root@swarm01 swarm]# docker stack ps myservice
ID                  NAME                  IMAGE               NODE                DESIRED STATE       CURRENT STATE                 ERROR               PORTS
qk8pvvc9nmv0        myservice_web.1       httpd:v2            swarm01             Running             Running 56 seconds ago
unwepav5z9nz         \_ myservice_web.1   httpd:v1            swarm02             Shutdown            Shutdown 48 seconds ago
uyj0pm1df9hl        myservice_web.2       httpd:v2            swarm02             Running             Running 46 seconds ago
3j5o1tktcg1l         \_ myservice_web.2   httpd:v1            swarm01             Shutdown            Shutdown about a minute ago
qdjov5zf5alb        myservice_web.3       httpd:v2            swarm03             Running             Running 10 seconds ago
wl1unt6lynft         \_ myservice_web.3   httpd:v1            swarm03             Shutdown            Shutdown 12 seconds ago
```

可以看到同时只有2个容器在升级，同时第3个容器也是在间隔了30s 之后，才开始升级。

 --filter 过滤条件`docker stack ps  --filter  "desired-state=running"  myservice`

### 4.6.6 服务健康检查

对于容器而言，最简单的健康检查是进程级的健康检查，即检验进程是否存活。Docker Daemon会自动监控容器中的PID1进程，如果docker run命令中指明了restart policy，可以根据策略自动重启已结束的容器。在很多实际场景下，仅使用进程级健康检查机制还远远不够。比如，容器进程虽然依旧运行却由于应用死锁无法继续响应用户请求，这样的问题是无法通过进程监控发现的 。

下面先介绍Docker容器健康检查机制，之后再介绍Docker Swarm mode的新特性。

**Docker 原生健康检查能力**

而自 1.12 版本之后，Docker 引入了原生的健康检查实现，可以在Dockerfile中声明应用自身的健康检测配置。HEALTHCHECK 指令声明了健康检测命令，用这个命令来判断容器主进程的服务状态是否正常，从而比较真实的反应容器实际状态。

HEALTHCHECK 指令格式：

- HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
- HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉

注：在Dockerfile中 HEALTHCHECK 只可以出现一次，如果写了多个，只有最后一个生效。

使用包含 HEALTHCHECK 指令的dockerfile构建出来的镜像，在实例化Docker容器的时候，就具备了健康状态检查的功能。启动容器后会自动进行健康检查。

> HEALTHCHECK 支持下列选项：
>
> - interval=<间隔>：两次健康检查的间隔，默认为 30 秒;
> - timeout=<间隔>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒;
> - retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
> - start-period=<间隔>: 应用的启动的初始化时间，在启动过程中的健康检查失效不会计入，默认 0 秒; (从17.05)引入

在 HEALTHCHECK [选项] CMD 后面的命令，格式和 ENTRYPOINT 一样，分为 shell 格式，和 exec 格式。命令的返回值决定了该次健康检查的成功与否：

- 0：成功;
- 1：失败;
- 2：保留值

容器启动之后，初始状态会为 starting (启动中)。Docker Engine会等待 interval 时间，开始执行健康检查命令，并周期性执行。如果单次检查返回值非0或者运行需要比指定 timeout 时间还长，则本次检查被认为失败。如果健康检查连续失败超过了 retries 重试次数，状态就会变为 unhealthy (不健康)。

注：

- 一旦有一次健康检查成功，Docker会将容器置回 healthy (健康)状态
- 当容器的健康状态发生变化时，Docker Engine会发出一个 health_status 事件。

假设我们有个镜像是个最简单的 Web 服务，我们希望增加健康检查来判断其 Web 服务是否在正常工作，我们可以用 curl来帮助判断，其 Dockerfile 的 HEALTHCHECK 可以这么写：

```
FROM elasticsearch:5.5 
 
HEALTHCHECK --interval=5s --timeout=2s --retries=12 \ 
  CMD curl --silent --fail localhost:9200/_cluster/health || exit 1
```

然后编译镜像，并运行

```
docker build -t test/elasticsearch:5.5 . 
docker run --rm -d \ 
    --name=elasticsearch \ 
    test/elasticsearch:5.5 
```

执行 docker ps容器检查命令，发现过了几秒之后，Elasticsearch容器从 starting 状态进入了 healthy 状态

```
$ docker ps 
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                            PORTS                NAMES 
c9a6e68d4a7f        test/elasticsearch:5.5   "/docker-entrypoin..."   2 seconds ago       Up 2 seconds (health: starting)   9200/tcp, 9300/tcp   elasticsearch 
$ docker ps 
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                    PORTS                NAMES 
c9a6e68d4a7f        test/elasticsearch:5.5   "/docker-entrypoin..."   14 seconds ago      Up 13 seconds (healthy)   9200/tcp, 9300/tcp   elasticsearch 
```

**Docker Swarm健康检查能力**

在Docker 1.13之后，在Docker Swarm mode中提供了对健康检查策略的支持。

可以在 `docker service create` 命令中指明健康检查策略

```
$ docker service create -d \
    --name=elasticsearch \
    --health-cmd="curl --silent --fail localhost:9200/_cluster/health || exit 1" \
    --health-interval=5s \
    --health-retries=12 \
    --health-timeout=2s \
    elasticsearch
```

在Swarm模式下，Swarm manager会监控服务task的健康状态，如果容器进入 `unhealthy` 状态，它会停止容器并且重新启动一个新容器来取代它。这个过程中会自动更新服务的 load balancer (routing mesh) 后端或者 DNS记录，可以保障服务的可用性。

在1.13版本之后，在服务更新阶段也增加了对健康检查的支持，这样在新容器完全启动成功并进入健康状态之前，load balancer/DNS解析不会将请求发送给它。这样可以保证应用在更新过程中请求不会中断。

![](.\img\healthcheck.png)

docker 官方网站有些常用容器的健康检查的例子：https://github.com/docker-library/healthcheck

# 5.附录

## 5.1 参考文档

官网文档：

https://docs.docker.com/engine/swarm/

https://docs.docker.com/get-started/part4/

https://docs.docker.com/network/overlay/#publish-ports-on-an-overlay-network

技术博客：

http://www.dockerinfo.net/2552.html

https://www.cnblogs.com/bigberg/p/8761047.html

http://blog.51cto.com/cloudman/1968287 

https://neuvector.com/network-security/docker-swarm-container-networking/

http://www.uml.org.cn/yunjisuan/201708282.asp