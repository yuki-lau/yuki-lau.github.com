---
layout: post
title: HDFS客户端读文件流程
tags: [Hadoop, HDFS]
category: HDFS
---

###摘要：
> 本文重点介绍HDFS中由客户端发起的读取文件操作，在这个流程中，主要可以分为三个阶段：
> 
> * 客户端与名字节点的交互：请求文件元数据（LocatedBlocks）、上报错误块；
> * 客户端内部处理：选择数据节点；
> * 客户端与数据节点的交互：读取数据。

###参考：
* <a href="http://book.douban.com/subject/24294210/" target="_blank">《Hadoop技术内幕：深入解析Hadoop Common和HDFS架构设计与实现原理》</a>

###目录：
* 整理流程
* 源码分析
	* 1.客户端创建文件输入流
	* 2.客户端与DataNode建立连接
	* 3.客户端从DataNode读取数据
	* 4.特殊处理：getBlockLocations()、reportBadBlocks()
	* 5.关闭文件数据流
* 异常处理

###整体流程
<img src="/images/201403/4.png" alt="客户端到名字节点的读取文件操作" />

如上图所示，客户端发起的读取文件的操作：

1. 客户端通过FileSystem.open()打开文件。对应HDFS具体文件系统，即调用DistributedFileSystem.open()方法。该最终创建一个DFSClient.DFSDataInputStream实例返回，客户后续则使用这个输入流进行数据的读取。
2. 在DFSDataInputStream构造函数中，调用了openInfo()方法，该方法通过ClientProtocol.getBlockLocations()远程接口调用NameNode，以确定文件开始部分数据块的保存位置。
3. 在获取DFSDataInputStream实例后，客户端就可以通过调用DFSDataInputStream.read()方法读取文件数据了。
4. DFSDataInputStream对象会通过和DataNode间的“读数据”流接口，和最近的DataNode建立联系。客户端反复调用read()方法，数据会通过DataNode和Client连接上的数据包返回客户端。
5. 当DFSDataInputStream.read()读到一个数据块的末端时，DFSDataInputStream会关闭和数据节点间的链接，并通过ClientProtocol.getBlockLocations()远程接口获得保存着当前文件的下一个数据块的数据节点信息。
6. 当客户端读完文件的所有数据后，通过DFSDataInputStream.close()关闭数据流。

> **注意**：
> 
> 在步骤2中，NameNode返回给客户端存储数据块的这些数据节点，已经根据它们与客户端的距离（利用了网络的拓扑信息）进行了简单的排序。
> 
> 在步骤5中，并不会每次都调用getBlockLocations()方法，因为DFSClient默认预取一个文件的10个LocatedBlock信息，所以只有在预取的blocks信息都读完时，才会再次调用getBlockLocations()方法。

以上就是HDFS中客户端读文件流程的大体过程了，这里没有涉及过多的源码细节，旨在对整体有一个清晰的了解，接下来我们将在源码层面上对读文件流程进行更加细致的介绍。

###源码分析

通过上面对读取文件流程的整体介绍，我们可以将读文件过程分为以下几个阶段：

1. 客户端创建文件输入流（同时获取文件开始数据的LocatedBlocks信息）；
2. 客户端与DataNode建立连接；
3. 客户端从DataNode读取数据；
4. 特殊处理：getBlockLocations()、reportBadBlocks()；
5. 读取完毕，关闭文件数据流。

####1. 客户端创建文件输入流

<img src="/images/201403/5.png" alt="DFSClient.open()序列图" />

如上图所示，为客户端创建文件输入流的序列图。客户端创建文件输入流的过程，即为调用FileSystem.open()方法的过程，这个方法最终通过调用DFSClient.open()创建DFSInputStream实例实现。

文件输入流DFSInputStream包含成员变量locatedBlocks，该对象类型为LocatedBlocks，包含当前输入流所要读取的文件中blocks的位置信息。因此在创建DFSInputStream实例的过程中，需要调用远程方法ClientProtocol.getBlockLocations()来获取文件包含的blocks信息。

若客户端目标读取的文件是一个正在创建中的文件的最后一块（LocatedBlocks.isUnderConstruction == true），则DFSClient还需要访问一次DataNode，请求文件的最新长度，并更新Block信息。

以上过程执行完毕后，一个文件输入流DFSInputStream就被创建完成并返回给客户端程序了。

####2. 客户端与DataNode建立连接

客户端获取文件输入流DFSInputStream实例后，就可以执行DFSInputStream.read()方法进行数据读取了，但在真正进行数据读取前，还需要创建于DataNode的连接，这个过程对于客户端程序来说是透明的。

在DFSInputStream.read()方法中，会首先检查当前读取的位置pos是否大于文件的长度getFileLength()，如果大于文件长度，则表明已经读到了块尾，需要调用**blockSeekTo()**来建立客户端与下一个数据块所在的DataNode的连接，并初始化读数据需要的**BlockReader**对象。

DFSInputStream.read()真正进行读数据的操作，也委托给 readBuffer(buf, off, realLen) 方法了，readBuffer中实际也是调用BlockReader方法进行读数据的。

<pre>
// 建立客户端与下一个数据块所在的DataNode的连接，并初始化读数据需要的BlockReader对象
private synchronized DatanodeInfo blockSeekTo(long target) throws IOException {
      ...
	// 关闭已有BlockReader和Socket连接
    if ( blockReader != null ) {
    	blockReader.close(); 
        blockReader = null;
    }
      
    if (s != null) {
        s.close();
        s = null;
    }

    // 获取下一个要读的Block信息
    LocatedBlock targetBlock = getBlockAt(target);
    
	...

    DatanodeInfo chosenNode = null;
    while (s == null) {
		// 选择存储targetBlock的DataNode
        DNAddrPair retval = chooseDataNode(targetBlock);
        chosenNode = retval.info;
        InetSocketAddress targetAddr = retval.addr;

        try {
			// 与DataNode建立连接
          	s = socketFactory.createSocket();
          	NetUtils.connect(s, targetAddr, socketTimeout);
          	s.setSoTimeout(socketTimeout);
          	Block blk = targetBlock.getBlock();
          
			// 创建并初始化读数据需要的BlockReader对象
          	blockReader = BlockReader.newBlockReader(s, src, blk.getBlockId(), 
            blk.getGenerationStamp(),
            offsetIntoBlock, blk.getNumBytes() - offsetIntoBlock,
            buffersize, verifyChecksum, clientName);
          	return chosenNode;
        } 
		catch (IOException ex) {
          	LOG.debug("Failed to connect to " + targetAddr + ":" 
                    + StringUtils.stringifyException(ex));
            
            // 选择DataNode时出现异常，将这个DataNode加入DeadNode列表
          	addToDeadNodes(chosenNode);
          	if (s != null) {
          		try {
              		s.close();
            	} catch (IOException iex) {}                        
          	}
          	s = null;
        }
    }
    return chosenNode;
}
</pre>

以上过程顺利执行完毕后，会返回与客户端建立连接的DataNodeInfo对象，且BlockReader初始化完毕，可用于进行数据流读取了。

> **注意**：在构造BlockReader的过程中，如果选择的DataNode就是本地节点的话，则创建的是BlockReader的子类**BlockReaderLocal**，以进行本地读取优化。

####3. 客户端从DataNode读取数据

BlockReader在读取数据时，会首先发作操作码为81的流式请求接口包。

DataNode的数据收发服务线程DataXceiverServer在收到请求后，会创建一个DataXceiver实例，并将请求转交给DataXceiver处理。

DataXceiver会根据请求OP的操作码进行判断，这里是读数据请求，因此将请求传递至readBlock()方法进行处理。在readBlock()方法中，会根据block id创建Block对象，并创建一个BlockSender实例用于将数据发送至客户端。

<pre>
private void readBlock(DataInputStream in) throws IOException {
	// 根据Block ID创建Block对象
    long blockId = in.readLong();          
    Block block = new Block( blockId, 0 , in.readLong());

    long startOffset = in.readLong();
    long length = in.readLong();
    String clientName = Text.readString(in);

    // 将Block数据放入输出流
    OutputStream baseStream = NetUtils.getOutputStream(s, 
        datanode.socketWriteTimeout);
    DataOutputStream out = new DataOutputStream(
                 new BufferedOutputStream(baseStream, SMALL_BUFFER_SIZE));
    
    BlockSender blockSender = null;
    ...
    try {
      	try {
			// 创建BlockSender
        	blockSender = new BlockSender(block, startOffset, length,
            	true, true, false, datanode, clientTraceFmt);
      	} catch(IOException e) {
        	out.writeShort(DataTransferProtocol.OP_STATUS_ERROR);
        	throw e;
      	}

      	out.writeShort(DataTransferProtocol.OP_STATUS_SUCCESS); 	// 发送操作码
      	long read = blockSender.sendBlock(out, baseStream, null); 	// 发送数据

      	...

    	} 
		...
    } 
	finally {
      	IOUtils.closeStream(out);
      	IOUtils.closeStream(blockSender);
    }
}
</pre>

####4. 特殊处理：getBlockLocations()、reportBadBlocks()

* getBlockLocations()：在客户端读取数据的过程中，可能会出现一次请求（默认10个LocatedBlock）的blocks不够读，需要再次请求NameNode获取文件后续block信息的情况。
* reportBadBlocks()：在客户端从DataNode读回的数据中，包含数据的Checksum信息，如果客户端本地校验失败，则会调用ClientProtocol.reportBadBlocks()报告坏块。

####5. 关闭文件数据流

应用读入所需的数据后，需要关闭输入流。

DFSInputStream.close()用于关闭流，这个方法很简单，检查了对象和所属的DFSClient对象的状态后，关闭可能打开的BlockReader对象和到数据节点的Socket对象，在调用了父类的close()方法后，设置标志位closed。

###异常处理
* 在客户端读取文件时，如果数据节点发生了错误，如节点停机或者网络出现故障，那么客户端会尝试下一个数据块的位置。同时，它也会记住出故障的那个数据节点（调用 **addToDeadNodes(DatanodeInfo info)**），不会再进行尝试。
* 在客户端读取文件时，如果数据块出现了错误，即校验和不一致，则说明数据块已经损坏，那么客户端会尝试从别的数据节点中读取另外一个副本文件的内容。同时，它也会将这个信息报告给（调用 **reportBadBlock()**）名字节点。

