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
	* **NamenodeProtocol**：Secondary NameNode、HDFS均衡器、NameNode之间的接口。由于调用方的是SecondaryNameNode、Balancer，因此由NameNode实现该接口。
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
HDFS的文件与目录操作一般是指客户端对HDFS文件目录树的操作，即客户端到名字节点的元数据操作，如更改文件名（rename），在给定目录下创建一个子目录（mkdir），删除某个文件（delete）等。

HDFS的文件目录树维护在NameNode中，而NameNode与DataNode的交互维持着主从关系。因此，这些文件与目录操作一般只涉及客户端和名字节点的交互，通过远程接口ClientProtocol进行，对元数据进行“标记”，至于“真正”对数据的操作，则需要在DataNode上报NameNode时，返回相应的DatanodeCommand指挥DataNode进行执行。

接下来，可以参阅 <a href="/hdfs/2014/03/05/client-delete-file-process-in-hdfs.html" target="_blank">HDFS客户端删除文件流程</a>，进行理解。

###HDFS读文件流程
参阅 <a href="/hdfs/2014/03/11/client-read-file-process-in-hdfs.html" target="_blank">HDFS客户端读文件流程</a>

###HDFS写文件流程
参阅 <a href="/hdfs/2014/04/11/client-write-file-process-in-hdfs.html" target="_blank">HDFS客户端写文件流程</a>