---
layout: post
title: HDFS客户端写文件流程
tags: [Hadoop, HDFS]
category: HDFS
---

###摘要：
> 本文重点介绍HDFS中由客户端发起的写文件操作，在这个流程中，主要可以分为三个部分：
> 
> * 客户端与名字节点的交互：创建文件、申请数据块；
> * 客户端与数据节点的交互：写数据；
> * 数据节点间的交互：数据流管道。

###参考：
* <a href="http://book.douban.com/subject/24294210/" target="_blank">《Hadoop技术内幕：深入解析Hadoop Common和HDFS架构设计与实现原理》</a>

###目录：
* 整理流程
* 异常处理

###整体流程
<img src="/images/201404/1.gif" alt="HDFS客户端写文件流程" />

如上图所示，客户端发起的写文件的操作：

1. 客户端调用DistributedFileSystem.create()方法创建文件，指明欲创建文件的文件名。
2. DistributedFileSystem继续创建DFSOutputStream，并由远程过程调用，让名字节点执行同名create方法，在文件系统的命名空间中创建一个新文件。远程方法调用结束后，DistributedFileSystem将该**DFSOutputStream**对象包裹在**FSDataOutputStream**实例中，返回给客户端。
4. 客户端写入FSDataOutputStream流中的数据，被分成包一个一个的文件包(packet), 放入一个中间队列**数据队列(DFSOutputStream.dataQueue)**中去。
5. 在步骤1、2中，create()调用创建了一个空文件，所以DFSOutputStream实例首先需要向名字节点申请数据块，调用**addBlock()**方法（图中步骤4），该方法执行成功后，返回一个LocatedBlock对象，该对象包含了新数据块的数据块标识和版本号，同时，它的成员变量**LocatedBlock.locs提供了数据流管道**的信息，通过这些信息DFSOutputStream就可以和数据节点联系，通过写数据接口建立数据流管道。
5. **DataStreamer**从dataQueue中取数据，打成数据包，写入到管道线中的第一个DataNode中（图中步骤5），第一个DataNode再把接收到的数据转到第二个DataNode中（图中步骤5），以此类推。
6. DFSOutputStream同时也维护着另一个中间队列**确认队列(DFSOutputStream.ackQueue)**，确认队列中的包只有在得到管道线中所有的DataNode的确认以后才会被移出确认队列。确认包逆流而上，从数据流管道依次发往客户端，当客户端收到应答时，它将对应的包从ackQueue移除。
7. DFSOutputStream在写完一个数据块后，数据流官道上的节点，会通过DatanodeProtocol远程接口的**blockReceived()**方法，向NameNode提交数据块。如果数据队列中还有等待输出的数据，DFSOutputStream对象需要再次调用addBlock()方法，为文件添加新的数据块。
8. 客户端完成数据的写入后，调用**close()**方法关闭流，意味着客户端**不会再往流中写数据**。
9. 当**dataQueue中的文件包都收到应答**后，就可以使用**ClientProtocol.complete()**方法通知名字节点关闭文件，完成一次正常的写文件流程。此时，NameNode就知道该文件由哪些block组成（因为DataStreamer向namenode请求分配新block，namenode当然会知道它分配过哪些blcok给当前文件），它会等待最少的replica数被创建，然后成功返回。
　　

###异常处理

如果在文件数据写入期间，数据节点发生故障，则会执行下面的操作：

1. 数据流管道会被关闭，已经发送到管道但还没有收到确认的文件包，会重新添加到dataQueue，这样保证了无论数据流管道中的哪个数据节点故障，都不会丢数据。
2. 当前正常运行的数据节点上的数据块会被赋予一个新的版本号，并通知名字节点。这样，失败的数据节点从故障中恢复过来以后，上面只有部分数据的数据块会因为数据块版本号和名字节点保存的版本号不匹配而被删除。
3. 在数据流管道中删除故障的数据节点，并重新建立管道，继续写数据到正常工作的数据节点。文件关闭后，名字节点会发现该数据块的副本数没有达到要求（即，处在under-replicated/备份数不足的状态），会选择一个新的数据节点并复制数据块，创建新的副本。

因此，数据节点故障只会影响一个数据块的写操作，后续数据块写入不会受影响。

> **注意：**
> 
> * 以上操作对于写入数据的客户端而言是透明的；
> * 对于出现多于1个数据节点出故障的情况（虽然不太经常发生），这时，只要数据流管道中的数据节点数满足配置项${dfs.replication.min}的值（默认为1），就认为写操作是成功的。后续这个数据块会被复制，直到满足文件副本系数的要求。

