---
layout:     post
title:      Dubbo & Dubbox
subtitle:   分布式RPC框架dubbo详解
date:       2017-08-10
author:     Nolan
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - dubbo
    - dubbox
---

## 一、初识Dubbo ##

**背景** ：随着互联网的发展，应用服务的规模不断扩大，常规的垂直服务架构已经无法应对，分布式服务架构以及流动计算架构势在必行，急需一个治理系统取保架构有条不紊的演进。

**认识**：Dubbo是阿里巴巴提供的开源的一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。

## 二、Dubbo工作原理-主体架构 ##

![dubbo架构设计](/img/dubbo_job_lc.jpg)
### Dubbo工作原理-核心部分 ###
**远程通讯**：提供对多种基于长连接的NIO框架抽象封装，包含多种线程
模型，序列化，以及“请求-响应”模式的信息交换方式。

**集群容错**：提供基于接口方法的透明远程过程调用，包括多协议支持，
以及软负载均衡，失败容错，地址路由，动态配置等集群支持。

**自动发现**：基于注册中心目录服务，试服务消费方能动态的查找服务提
供方，使地址透明，使服务提供方可以平滑增加或减少机器。

### Dubbo功能特性-RPC功能 ###
![dubbo RPC功能](/img/dubbo_rpc.jpg)
### Dubbo功能特性-健壮性 ###
1、监控中心宕掉不影响使用，只是丢失部分采样数据

2、数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务

3、注册中心对等集群，任意一台宕掉后，将自动切换到另一台

4、注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯

5、服务提供者无状态，任意一台宕掉后，不影响使用

6、服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

### Dubbo功能特性-伸缩性 ###
注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心

服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者

## 三、Dubbo快速入门案例 ##

### 快速启动 ###
Dubbo采用全Spring配置方式，透明化接入应用，对应用没有任何API侵入，只需用Spring加载Dubbo的配置即可，Dubbo基于Spring的Schema扩展进行加载。

如果不想使用Spring配置，而希望通过API的方式进行调用（不推荐）

### 服务提供者-定义服务接口 ###

(注：该接口需单独打包，在服务提供方和消费方共享)
![定义接口](/img/dubbo_dyjk.jpg)
### 服务提供者-实现接口 ###
(注：对服务消费方隐藏实现)
![接口实现](/img/dubbo_jksx.jpg)
### 服务提供者-Spring配置声明暴露服务 ###
![暴露服务](/img/dubbo_plfw.jpg)
### 服务提供者-加载配置 ###
![启动服务](/img/dubbo_qdfw.jpg)
### 服务消费者-通过spring配置引用远程服务 ###
![消费服务](/img/dubbo_xffw.jpg)

## 四、服务治理 ##

![服务治理](/img/dubbo_fwzl.png)

### 各节点关系 ###

这里的Invoker是Provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息。

Directory代表多个Invoker，可以把它看成List<Invoker>，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。

Cluster将Directory中的多个Invoker伪装成一个Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个。

Router负责从多个Invoker中按路由规则选出子集，比如读写分离，应用隔离等。

LoadBalance负责从多个Invoker中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选。

### 集群容错模式 ###

**Failover Cluster**
失败自动切换，当出现失败，重试其它服务器。(缺省)
通常用于读操作，但重试会带来更长延迟。
可通过retries="2"来设置重试次数(不含第一次)。

**Failfast Cluster**
快速失败，只发起一次调用，失败立即报错。
通常用于非幂等性的写操作，比如新增记录。

**Failsafe Cluster**
失败安全，出现异常时，直接忽略。
通常用于写入审计日志等操作。

**Failback Cluster**
失败自动恢复，后台记录失败请求，定时重发。
通常用于消息通知操作。

**Forking Cluster**
并行调用多个服务器，只要一个成功即返回。
通常用于实时性要求较高的读操作，但需要浪费更多服务资源。
可通过forks="2"来设置最大并行数。

**Broadcast Cluster**
广播调用所有提供者，逐个调用，任意一台报错则报错。(2.1.0开始支持)
通常用于通知所有提供者更新缓存或日志等本地资源信息。

### 负载均衡 ###

**Random LoadBalance**
随机，按权重设置随机概率。
在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

**RoundRobin LoadBalance**
轮循，按公约后的权重设置轮循比率。
存在慢的提供者累积请求问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

**LeastActive LoadBalance**
最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

**ConsistentHash LoadBalance**
一致性Hash，相同参数的请求总是发到同一提供者。
当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
算法参见：http://en.wikipedia.org/wiki/Consistent_hashing。
缺省只对第一个参数Hash，如果要修改，请配置<dubbo:parameter key="hash.arguments" value="0,1" />
缺省用160份虚拟节点，如果要修改，请配置<dubbo:parameter key="hash.nodes" value="320" />

## 五、RPC ##

RPC是指远程过程调用，也就是说两台服务器A，B，一个应用部署在A服务器上，想要调用B服务器上提供的函数/方法，由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。

### RPC核心 ###

动态代理和Socket

### RPC性能 ###

**传输**：用什么样的通道将数据发送给对方，BIO、NIO或者AIO，IO模型在很大程度上决定了框架的性能。 
      I/O调度模型：同步阻塞I/O（BIO）还是非阻塞I/O（NIO）。

**协议**：采用什么样的通信协议，HTTP或者内部私有协议。协议的选择不同，性能模型也不同。相比于公有协议，内部私有协议的性能通常可以被设计的更优。 
      序列化框架的选择：文本协议、二进制协议或压缩二进制协议。

**线程**：数据报如何读取？读取之后的编解码在哪个线程进行，编解码后的消息如何派发，Reactor线程模型的不同，对性能的影响也非常大。 
      线程调度模型：串行调度还是并行调度，锁竞争还是无锁化算法。

### 分布式RPC框架有哪些 ###

**Dubbo** 
Motan是新浪微博开源的一个Java 框架。它诞生的比较晚，起于2013年，2016年5月开源。

**Motan** 在微博平台中已经广泛应用，每天为数百个服务完成近千亿次的调用。

**rpcx** 是Go语言生态圈的Dubbo， 比Dubbo更轻量，实现了Dubbo的许多特性，借助于Go语言优秀的并发特性和简洁语法，可以使用较少的代码实现分布式的RPC服务。

**gRPC** 是Google开发的高性能、通用的开源RPC框架，其由Google主要面向移动应用开发并基于HTTP/2协议标准而设计，基于ProtoBuf(Protocol Buffers)序列化协议开发，且支持众多开发语言。本身它不是分布式的，所以要实现上面的框架的功能需要进一步的开发。
thrift是Apache的一个跨语言的高性能的服务框架，也得到了广泛的应用。

## 六、DubboX新特性 ##

支持REST风格远程调用（HTTP + JSON/XML)：基于非常成熟的JBoss RestEasy框架，在dubbo中实现了REST风格（HTTP + JSON/XML）的远程调用，以显著简化企业内部的跨语言交互，同时显著简化企业对外的Open API、无线API甚至AJAX服务端等等的开发。事实上，这个REST调用也使得Dubbo可以对当今特别流行的“微服务”架构提供基础性支持。 另外，REST调用也达到了比较高的性能，在基准测试下，HTTP + JSON与Dubbo 2.x默认的RPC协议（即TCP + Hessian2二进制序列化）之间只有1.5倍左右的差距，详见文档中的基准测试报告。

支持基于Kryo和FST的Java高效序列化实现：基于当今比较知名的Kryo和FST高性能序列化库，为Dubbo默认的RPC协议添加新的序列化实现，并优化调整了其序列化体系，比较显著的提高了Dubbo RPC的性能，详见文档中的基准测试报告。

支持基于Jackson的JSON序列化：基于业界应用最广泛的Jackson序列化库，为Dubbo默认的RPC协议添加新的JSON序列化实现。

支持基于嵌入式Tomcat的HTTP remoting体系

升级Spring：将dubbo中Spring由2.x升级到目前最常用的3.x版本，减少版本冲突带来的麻烦。

升级ZooKeeper客户端：将dubbo中的zookeeper客户端升级到最新的版本，以修正老版本中包含的bug。

支持完全基于Java代码的Dubbo配置：基于Spring的Java Config，实现完全无XML的纯Java代码方式来配置dubbo

调整Demo应用：暂时将dubbo的demo应用调整并改写以主要演示REST功能、Dubbo协议的新序列化方式、基于Java代码的Spring配置等等。
修正了dubbo的bug 包括配置、序列化、管理界面等等的bug。

### 快速入门 ###

![](/img/dubbox_1.png)

![](/img/dubbox_2.png)

![](/img/dubbox_3.png)

### Dubbox使用场景 ###

非dubbo的消费端调用dubbo的REST服务（non-dubbo --> dubbo）

dubbo消费端调用dubbo的REST服务 （dubbo --> dubbo）

dubbo的消费端调用非dubbo的REST服务 （dubbo --> non-dubbo）

## 结束 ## 


