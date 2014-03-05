---
layout: post
title: HDFS通信接口和文件读写流程
tags: [Hadoop, HDFS]
category: HDFS
---

###摘要：
> 本文重点介绍HDFS中NameNode、DataNode和Client这3个主要角色之间的通信方式：基于IPC接口的通信和基于TCP或HTTP的流式接口通信。并分别介绍基于IPC接口的HDFS文件与目录操作流程，基于流式接口的HDFS读文件流程、写文件流程。

###参考：
* <a href="http://book.douban.com/subject/24294210/" target="_blank">《Hadoop技术内幕：深入解析Hadoop Common和HDFS架构设计与实现原理》</a>

###目录：
* 通信接口
	* 基于IPC的接口
	* 基于TCP或HTTP的流式接口
* HDFS文件与目录操作流程
* HDFS读文件流程
* HDFS写文件流程

###通信接口
HDFS的体系结构中包含了NameNode、DataNode和Client这3个主要角色，它们间有两种主要的通信接口：

* Hadoop远程过程调用（Inter-Process Communication, IPC）接口
* 基于TCP或HTTP的流式接口

####基于IPC的接口
基于IPC的接口主要分为三类：

* 客户端接口：
	* **ClientProtocol**：Client与NameNode之间的接口。由于调用方是Client，因此是NameNode实现该接口；
	* **ClientDatanodeProtocol**：Client与DataNode之间的接口。由于调用方是Client，因此是DataNode实现该接口。该接口用的比较少，因为Client与DataNode间主要是流数据传输，只是在发生错误时需要通过该IPC接口获取一些信息配合进行恢复。
* 服务器间的接口：
	* **DatanodeProtocol**：DataNode与NameNode之间的接口。由于调用方是DataNode，因此是NameNode实现该接口。DataNode不断通过这个接口向NameNode报告一些信息，方法的返回值会带回名字节点指令。
	* **InterDatanodeProtocol**：DataNode与DataNode之间的接口。由于是DataNode之间互相调用，因此是DataNode实现该接口。
	* **NamenodeProtocol**：Secondary NameNode、HDFS均衡器、NameNode之间的接口。由于调用的是NameNode，因此由NameNode实现该接口。
* 安全相关的接口
	* **RefreshAuthorizationPolicyProtocol**：NameNode实现该接口。

例如，NameNode类实现接口如下：
<pre>
public class NameNode implements ClientProtocol, DatanodeProtocol,
                                 NamenodeProtocol, FSConstants,
                                 RefreshAuthorizationPolicyProtocol {
	...
}
</pre>

**注意**：在各个远程调用之间，Hadoop IPC**不会在服务器端保存会话信息**，所以，调用一些需要根据当前上下文做操作的方法时（如addBlock()），**客户端需要提供上下文信息**（clientName）。

####基于TCP或HTTP的流式接口
流式接口主要用于大数据块传输，其中：

* 基于TCP：Client和DataNode、DataNode和DataNode间与数据块操作相关的接口；
* 基于HTTP：第二名字节点合并命名空间镜像和镜像编辑日志相关的接口。

###HDFS文件与目录操作流程
TODO

###HDFS读文件流程
TODO

###HDFS写文件流程
TODO