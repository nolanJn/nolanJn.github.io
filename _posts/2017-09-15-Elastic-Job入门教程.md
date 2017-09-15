---
layout:     post
title:      Elastic-Job
subtitle:   分布式定时任务框架
date:       2017-09-15
author:     Nolan
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - Job
    - 当当
    - 分布式
    - java
    - zookeeper
---

## 目录 ##

### 1、何为分布式作业 ###
### 2、项目结构说明 ###
### 3、如何使用 ###
### 4、代码开发指南 ###
### 5、运维平台 ###
### 6、阅读源码编译问题 ###
### 7、实现原理 ###
### 8、作业分片策略 ###
### 9、作业运行状态监控 ###

## 1、何为分布式作业 ##

**1.分片概念**

     任务的分布式执行，需要将一个任务拆分为n个独立的任务项，然后由分布式的服务器分别执行某一个或几个分片项。
例如：有一个遍历数据库某张表的作业，现有2台服务器。为了快速的执行作业，那么每台服务器应执行作业的50%。 为满足此需求，可将作业分成2片，每台服务器执行1片。作业遍历数据的逻辑应为：服务器A遍历ID以奇数结尾的数据；服务器B遍历ID以偶数结尾的数据。 如果分成10片，则作业遍历数据的逻辑应为：每片分到的分片项应为ID%10，而服务器A被分配到分片项0,1,2,3,4；服务器B被分配到分片项5,6,7,8,9，直接的结果就是服务器A遍历ID以0-4结尾的数据；服务器B遍历ID以5-9结尾的数据。

**2.分片项与业务处理解耦**

 Elastic-job并不直接提供数据处理的功能，框架只会将分片项分配至各个运行中的作业服务器，开发者需要自行处理分片项与真实数据的对应关系。

**3.分布式作业的执行**

     Elastic-job并无作业调度中心节点，而是基于部署作业框架的程序在到达相应时间点时各自触发调度。
注册中心仅用于作业注册和监控信息存储。而主作业节点仅用于处理分片和清理等功能。

**4.个性化参数的适用场景**

     个性化参数即shardingItemParameters，可以和分片项匹配对应关系，用于将分片项的数字转换为更加可读的业务代码。
例如：按照地区水平拆分数据库，数据库A是北京的数据；数据库B是上海的数据；数据库C是广州的数据。 如果仅按照分片项配置，开发者需要了解0表示北京；1表示上海；2表示广州。 合理使用个性化参数可以让代码更可读，如果配置为0=北京,1=上海,2=广州，那么代码中直接使用北京，上海，广州的枚举值即可完成分片项和业务逻辑的对应关

**5.作业高可用**

**6.最大限度利用资源**

## 2、项目结构说明 ##

![](/img/elastic-job-1.png)

	 1.1.1版本项目结构                       2.0.0版本项目结构
![](/img/elastic-job-2.png)       ![](/img/elastic-job-3.png)

## 3、使用步骤 ##

**环境准备**

1. Java环境
     JDK1.7及以上版本
2. Zookeeper
   Zookeeper 3.4.6及其以上版本。或使用Elastic-Job内嵌zookeeper
3. Maven 
     maven3.0.4及其以上版本
4. 引入elastic-job
	
		<dependency>
			<groupId>com.dangdang</groupId>
			<artifactId>elastic-job-core</artifactId>
			<version>2.0.0</version>
		</dependency>
	
		<dependency>
			<groupId>com.dangdang</groupId>
			<artifactId>elastic-job-spring</artifactId>
			<version>2.0.0</version>
		</dependency>

## 4、代码开发指南 ##

### 分三种作业类型：Simple  、 DataFlow  、 Script###

**Simple**

Simple类型作业意为简单实现，未经任何封装的类型。需要继承AbstractSimpleElasticJob，该类只提供了一个方法用于覆盖，此方法将被定时执行。用于执行普通的定时任务，与Quartz原生接口相似，只是增加了弹性扩缩容和分片等功能。

![](/img/elastic-job-4.png)

**DataFlow**

DataFlow类型用于处理数据流，它又提供2种作业类型，分别是ThroughputDataFlow和SequenceDataFlow。需要继承相应的抽象类。

1、ThroughputDataFlow

![](/img/elastic-job-5.png)

2、SequenceDataFlow  
   使用方法相同，继承： AbstractIndividualSequenceDataFlowElasticJob<T>

**Script**

注：批量处理
        为了提高数据处理效率，数据流类型作业提供了批量处理数据的功能。之前逐条处理数据的两个抽象类分别是AbstractIndividualThroughputDataFlowElasticJob和AbstractIndividualSequenceDataFlowElasticJob，
批量处理则使用另外两个接口AbstractBatchThroughputDataFlowElasticJob和AbstractBatchSequenceDataFlowElasticJob。不同之处在于processData方法的返回值从boolean类型变为int类型，用于表示一批数据处理的成功数量，第二个入参则转变为List数据集合。

![](/img/elastic-job-6.png)

## 5、运维平台 ##

配置：两台Zookeeper虚拟机，作业服务器从1台增加到3台

###一、单台作业服务器的情况：###

**运维平台：**

![](/img/elastic-job-7.png)

**Job运行情况**

![](/img/elastic-job-8.png)

**注册中心节点情况**

![](/img/elastic-job-9.png)

###二、3台作业服务器的情况：###

**运维平台：**

![](/img/elastic-job-10.png)

**Job运行情况**

![](/img/elastic-job-11.png)

**注册中心节点情况**

![](/img/elastic-job-12.png)

###三、补充说明：###

![](/img/elastic-job-13.png)

触发： 没测试出来干啥用的！！

暂停：点下暂停后按钮会变成[恢复]。暂停当前作业服务器的作业的运行，恢复后马上执行一次，暂停是错过    的不会再执行

关闭：服务器断开连接，该服务器上的分片会在下次作业执行前，重新分片到别的服务器上；

失效：对应服务器节点下产生disabled节点，下次作业前重新分片。点击生效后重新加入，重新分片。

## 6、实现原理 ##

弹性分布式实现

第一台服务器上线触发主服务器选举。主服务器一旦下线，则重新触发选举，选举过程中阻塞，只有主服务器选举完成，才会执行其他任务。

某作业服务器上线时会自动将服务器信息注册到注册中心，下线时会自动更新服务器状态。

3.    主节点选举，服务器上下线，分片总数变更均更新重新分片标记。

4.    定时任务触发时，如需重新分片，则通过主服务器分片，分片过程中阻塞，分片结束后才可执行任务。如分片过程中主服务器下线，则先选举主服务器，再分片。

5.    通过上一项说明可知，为了维持作业运行时的稳定性，运行过程中只会标记分片状态，不会重新分片。分片仅可能发生在下次任务触发前。

6.    每次分片都会按服务器IP排序，保证分片结果不会产生较大波动。

7.    实现失效转移功能，在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。

注册中心数据结构：

        注册中心在定义的命名空间下，创建作业名称节点，用于区分不同作业，所以作业一旦创建则不能修改作业名称，如果修改名称将视为新的作业。作业名称节点下又包含4个数据子节点，分别是config, servers, execution和leader。

## 7、分片策略 ##

1、AverageAllocationJobShardingStrategy

全路径：
com.dangdang.ddframe.job.plugin.sharding.strategy.AverageAllocationJobShardingStrategy

![](/img/elastic-job-14.png)

2、OdevitySortByNameJobShardingStrategy

全路径：
com.dangdang.ddframe.job.plugin.sharding.strategy.OdevitySortByNameJobShardingStrategy

![](/img/elastic-job-15.png)

**两种分片策略优缺点对比:**
AverageAllocationJobShardingStrategy的缺点是，一旦分片数小于作业服务器数，作业将永远分配至IP地址靠前的服务器，导致IP地址靠后的服务器空闲。而OdevitySortByNameJobShardingStrategy则可以根据作业名称重新分配服务器负载。如：
如果有3台服务器，分成2片，作业名称的哈希值为奇数，则每台服务器分到的分片是：1=[0], 2=[1], 3=[]
如果有3台服务器，分成2片，作业名称的哈希值为偶数，则每台服务器分到的分片是：3=[0], 2=[1], 1=[]

## 8、作业监控 ##

**作业运行状态监控**
通过监听elastic-job的zookeeper注册中心的几个关键节点即可完成作业运行状态监控功能。

**监听作业服务器存活:**
监听job_name\servers\ip_address\status节点是否存在。该节点为临时节点，如果作业服务器下线，该节点将删除。

**监听近期数据处理成功:**
数据流类型作业，可通过监听近期数据处理成功数判断作业流量是否正常。
监听job_name\servers\ip_address\processSuccessCount节点的值。如果小于作业正常处理的阀值，可选择报警。

**监听近期数据处理失败:**
数据流类型作业，可通过监听近期数据处理失败数判断作业处理结果。
监听job_name\servers\ip_address\processFailureCount节点的值。如果大于0，可选择报警。

## 结束 ## 


