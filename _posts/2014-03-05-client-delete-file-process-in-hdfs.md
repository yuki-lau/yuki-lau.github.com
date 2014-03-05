---
layout: post
title: HDFS客户端删除文件流程
tags: [Hadoop, HDFS]
category: HDFS
---

###摘要：
> 本文重点介绍HDFS中由客户端发起文件删除操作，进而名字节点处理删除请求，直至数据节点最终删除文件数据块的完整流程。在这个流程中，主要可以分为两个阶段：
> 
> * 客户端与名字节点的交互：删除文件元数据，并未涉及数据节点；
> * 数据节点与名字节点的交互：删除文件数据块，由数据节点上报（如心跳）名字节点，名字节点返回DatanodeCommand引起，并非名字节点主动联系数据节点。

###参考：
* <a href="http://book.douban.com/subject/24294210/" target="_blank">《Hadoop技术内幕：深入解析Hadoop Common和HDFS架构设计与实现原理》</a>

###目录：
* 整理流程
* 源码分析
	* 1.客户端发起删除文件操作
	* 2.名字节点处理删除操作远程调用
	* 3.名字节点生成删除数据块DatanodeCommand
	* 4.数据节点删除文件数据块

###整体流程
<img src="/images/201403/3.png" alt="客户端到名字节点的删除文件操作" />

如上图所示，客户端发起文件删除操作：

1. 调用DistributedFilesystem.delete()方法，该方法最终调用ClientProtocol接口中的delete()方法，即调用NameNode.delete()方法。
2. 名字节点执行delete()方法时，它只会标记操作涉及的需要被删除的数据块，并且记录delete操作持久化到EditLog。名字节点在此操作期间，并不会主动联系保存这些数据块的数据节点，即不会立即删除数据。这点很好理解，NameNode和DataNode维持着主从关系，NameNode绝不会主动调用DataNode的。
3. 当保存着这些数据块的数据节点向名字节点发送心跳时，在心跳的应答里，名字节点会通过DatanodeCommand命令数据节点删除数据。

这里有几个注意点：

* 被删除文件的元数据，也就是该文件在目录树上的信息，在删除操作完成后，会被立刻删除，客户端之后就无法从NameNode上请求到该文件了；
* 被删除文件的数据，也就是该文件对应的数据块，在删除操作完成的一段时间后，才会被真正删除；
* 名字节点和数据节点永远维持着简单的主从关系，名字节点不会向数据节点发起任何IPC调用，数据节点需要配合名字节点执行的操作，都是通过数据节点心跳应答中携带的DatanodeCommand返回的。

###源码分析
通过上面整体流程的介绍，我们可以很清晰的将文件删除操作分为两个阶段：

1. 删除文件元数据
2. 删除文件数据

更加详细的，可以将这两个阶段继续划分为四个阶段，以便于我们进行源码分析：

1. 客户端发起删除文件操作；
2. 名字节点处理删除操作远程调用：删除文件源数据，“标记”文件数据块为删除；
3. 名字节点生成删除数据块DatanodeCommand：ReplicationMonitor将“标记”删除的文件数据块生删除数据块的命令；
4. 数据节点删除文件数据块：数据及诶单上报心跳，携带回删除数据块命令，真正删除文件数据。

####1. 客户端发起删除文件操作
客户端程序通过调用DistributedFileSystem这类的实例来实现对HDFS的操作。Hadoop中定义了抽象文件接口 <code>org.apache.hadoop.fs.FileSystem</code>，DistributedFileSystem就是FileSystem的一个具体实现，也就是HDFS(Hadoop Distributed File System)这个名称的由来。

当客户端需要删除文件时，他调用DistributedFileSystem中的delete方法，DistributedFileSystem的delete方法只是简单调用了dfs.delete，dfs这个对象的类型为DFSClient，DistributedFileSystem中的大部分方法的具体实现都是由DFSClient实现的。在DFSClient.delete方法中会检测当前文件系统是否为running状态，如果非running则抛出异常，否则调用远程方法ClientProtocol.delete方法，即NameNode实现的delete方法，执行删除操作。
<pre>
// DistributedFileSystem的delete方法
public boolean delete(Path f, boolean recursive) throws IOException {
	return dfs.delete(getPathName(f), recursive);
}

// DFSClient的delete方法
public boolean delete(String src, boolean recursive) throws IOException {
	checkOpen();
    try {
    	return namenode.delete(src, recursive);
    } catch(RemoteException re) {
    	throw re.unwrapRemoteException(AccessControlException.class);
    }
}
</pre>
客户端的代码比较简单，到这里就介绍完了。接下来，删除操作由IPC方法namenode.delete()传递到了名字节点。

####2. 名字节点处理删除操作远程调用
名字节点处理删除操作远程调用主要实现两个目的：

1. 目标文件的元数据删除
2. 将目标文件对应的数据块放入recentInvalidateSets

整个过程涉及到了多个类和方法间的调用，可以参考下面的时序图，从而有一个整体的认识，再继续看下文的具体解释。

<img src="/images/201403/4.jpg" alt="名字节点处理删除操作远程调用时序图" />

名字节点NameNode实现了ClientProtocol接口，在接口方法 <code>public boolean delete(String src, boolean recursive) throws IOException</code> 中，简单检查了HDFS的状态后，即调用了namesystem.delete方法来实现删除操作。

> namesystem的类型为FSNamesystem，FSNamesystem是一个非常复杂的类，它是整个名字节点的门面，实现了NameNode大部分的具体操作。

FSNamesystem.delete()在进行参数检查后，调用其内部实现的deleteInternal()方法来完成具体的删除操作。需要说明的是，在FSNamesystem中，很多文件目录操作都对应一个xxxInternal方法，其中原始的xxx操作方法添加参数检查和日志同步。

FSNamesystem.deleteInternal()在进行HDFS状态检查、权限检查后，调用dir.delete操作，将删除传递到FSDirectory。

> dir的类型是FSDiretory，FSDirectory知晓设计目录树的操作如何分派到子系统的各个对象中，并实现响应的处理逻辑。它有60多个成员函数，基本上远程接口ClientProtocol中的方法，都能在FSDirectory找到对应。

FSDirectory.delete()在保证系统准备好后，调用FSDirectory.unprotectedDelete()，并在unprotectedDelete()成功后添加删除日志。

FSDirectory.unprotectedDelete()是真正删除文件元数据的地方了（呼~ HDFS为了保证系统的可靠性和灵活性，层层嵌套了多层实现和检查），在该方法中：

1. 首先根据文件路径获取该文件所在的INode数组（即这个文件路径上的所有目录对象和它本身这个文件对象），数组的最后一个inode对象为targetNode，即要删除的文件对象；
2. 将文件从目录对象中删除，并更新目录的修改时间；
3. 将文件对象对应的数据块blocks回收；
4. 调用Namesystem.removePathAndBlocks(src, v)将文件对应的数据块从元数据中删除（上文所提到的“标记”删除）；
5. 返回targetNode。

至此，目标文件对应的inode对象和其占用的blocks已经被找到并回收了，但是blocks与DataNode的对应关系还未在名字节点中删除，这个工作就是由Namesystem.removePathAndBlocks()来实现的。

Namesystem.removePathAndBlocks()的代码很简单：
<pre>
void removePathAndBlocks(String src, List<Block> blocks) throws IOException {
	leaseManager.removeLeaseWithPrefixPath(src);
    for(Block b : blocks) {
    	blocksMap.removeINode(b);
      	corruptReplicas.removeFromCorruptReplicasMap(b);
      	addToInvalidates(b);
    }
}
</pre>
如代码所示，该方法首先将租约记录从租约管理器LeaseManager中删除，然后将文件的每个block从blockMap、corruptReplicas中删除，并通过addToInvalidate()添加至recentInvalidateSets。

> **blocksMap**的类型为BlocksMap，它管理着数据节点上数据块的元数据，即block所属的文件是什么？block放在了哪些数据节点上？这类问题。
> 
> **corruptReplicas**的类型为CorruptReplicasMap，它存储着HDFS中所有损坏的数据块的信息，仅当一个数据块的所有副本都损坏时才认为这个数据块是损坏的，这个时候就需要从系统中删除或做适当的修复工作。
> 
> **recentInvalidateSets**的类型为Map<String, Collection<Block>>，其中，key为DatanodeDescriptor.storageID，value为需要被删除的数据块集合。
> 
> **DatanodeDescriptor为DataNode在NameNode上的抽象**，它与BlocksMap一起保存名字节点中数据块与数据节点映射的关系。

通过上面的分析，我们看到ClientProtocol.delete()将目标文件的元数据删除，并将目标文件对应的数据块放入了recentInvalidateSets中，至此它的任务就结束了。

剩下的事儿，就交给ReplicationMonitor来处理recentInvalidateSets，生成删除数据块命令下方给数据节点了。

####3. 名字节点生成删除数据块DatanodeCommand
复制线程ReplicationMonitor为FSNamesystem的内部类，它在运行的过程中不断的做两件事情的检查：

* computeDatanodeWork()：计算需要调度数据节点进行复制、删除的数据块，这些数据节点会在它们的下一次心跳上报时被通知到；
* processPendingReplication()：检查是否有数据块复制请求超时，如果有将这次复制过程撤销，并将数据块放回neededReplication队列，等待下一轮computeDatanodeWork重新生成复制请求。

由上描述可知，生成删除数据块命令的操作将在computeDatanodeWork()方法中完成。在computeDatanodeWork()方法中，又包含两个任务：

* computeReplicationWork()：计算需要复制的数据块 
* computeInvalidateWork()：计算需要删除的数据块

computeInvalidateWork()会根据需要处理的数据节点数量循环调用invalidateWorkForOneNode()，每次处理一个DataNode需要删除的数据块，将需要删除的数据块添加到存储它们的数据节点抽象DatanodeDescriptor中：
<pre>
private synchronized int invalidateWorkForOneNode() {
	...

    // 获取第一个需要处理无效数据块的数据节点
    String firstNodeId = recentInvalidateSets.keySet().iterator().next();
    assert firstNodeId != null;
	
	// 获取这个数据节点对应的DatanodeDescriptor对象
    DatanodeDescriptor dn = datanodeMap.get(firstNodeId);
    if (dn == null) {
       	removeFromInvalidates(firstNodeId);
       	return 0;
    }

	// 获取当前需要处理的数据节点上存储的失效的数据块集合
    Collection<Block> invalidateSet = recentInvalidateSets.get(firstNodeId);
    if(invalidateSet == null)
      	return 0;

	// 每次处理的失效block有数量限制
	// 根据blockInvalidateLimit限制创建blocksToInvalidate列表，并添加失效block到blocksToInvalidate中
    ArrayList<Block> blocksToInvalidate = 
      new ArrayList<Block>(blockInvalidateLimit);

    // # blocks that can be sent in one message is limited
    Iterator<Block> it = invalidateSet.iterator();
    for(int blkCount = 0; blkCount < blockInvalidateLimit && it.hasNext();
                                                                blkCount++) {
      blocksToInvalidate.add(it.next());
      it.remove();
    }

    // 如果当前需要处理的失效block数量没有超过blockInvalidateLimit，则表明当前数据节点上的失效block添加完毕
	// 将当前数据节点从invalidateSets中删除
    if (!it.hasNext()) {
      removeFromInvalidates(firstNodeId);
    }

	// **将需要删除的失效Block添加到DatanodeDescriptor对象**
    dn.addBlocksToBeInvalidated(blocksToInvalidate);

    ...

    return blocksToInvalidate.size();
}
</pre>

至此ReplicationMonitor针对删除数据块的工作就处理完毕了，可是我们发现此时还是没有生成删除数据节点命令，让我们注意上述代码 <code>dn.addBlocksToBeInvalidated(blocksToInvalidate);</code>， 我们说过DatanodeDescriptor是数据节点在名字节点上的抽象，每个数据节点在心跳上报到名字节点时，名字节点都会根据其对应的DatanodeDescriptor对象中的信息生成相关的操作命令DatanodeCommand返回给数据节点。

因此，当执行DatanodeDescriptor.addBlocksToBeInvalidated()方法时，失效的blocks就被放到了DatanodeDescriptor的成员变量invalidateBlocks中了。这样等到下次心跳上报时，就可以根据invalidateBlocks生成数据块删除指令了。

当数据节点上报心跳时，调用DatanodeProtocol.sendHeartbeat方法，该方法实现在了NameNode中，如下所示。可以看到，返回结果是一个DatanodeCommand数组。
<pre>
public DatanodeCommand[] sendHeartbeat(DatanodeRegistration nodeReg,
                                       	long capacity,
                                       	long dfsUsed,
                                       	long remaining,
                                       	int xmitsInProgress,
                                       	int xceiverCount) throws IOException {
	verifyRequest(nodeReg);
    return namesystem.handleHeartbeat(nodeReg, capacity, dfsUsed, remaining,
        xceiverCount, xmitsInProgress);
}
</pre>

在namesystem.handleHeartbeat()方法中，会获取当前上报心跳的数据节点的DatanodeDescriptor对象，并针对DatanodeDescriptor对象中的心跳生成相应的DatanodeCommand对象，如下：
<pre>
DatanodeCommand[] handleHeartbeat(DatanodeRegistration nodeReg,
      									long capacity, long dfsUsed, long remaining,
      									int xceiverCount, int xmitsInProgress) throws IOException {
    DatanodeCommand cmd = null;
    synchronized (heartbeats) {
      	synchronized (datanodeMap) {
			// 获取数据节点对应的DatanodeDescriptor对象
        	DatanodeDescriptor nodeinfo = null;
        	try {
          		nodeinfo = getDatanode(nodeReg);
        	} catch(UnregisteredDatanodeException e) {
          		return new DatanodeCommand[]{DatanodeCommand.REGISTER};
        	}

          	...
        
			// 检查失效block，生成删除数据块DatanodeCommand，并添加到返回命令数组中
        	cmd = nodeinfo.getInvalidateBlocks(blockInvalidateLimit);
        	if (cmd != null) {
          		cmds.add(cmd);
        	}
        	if (!cmds.isEmpty()) {
          		return cmds.toArray(new DatanodeCommand[cmds.size()]);
        	}
      	}
    }

    ...

    return null;
}
</pre>
虽然在说明上述问题的过程中，用了比较多的代码，显得略微复杂，但实际上就两点：

1. ReplicationMonitor定期从recentInvalidateSets中收集并计算需要删除的数据块，并将这些数据块放到对应DatanodeDescriptor对象的invalidateBlocks中；
2. 数据节点上报心跳时，名字节点会根据数据节点对应的DatanodeDescriptor中的invalidateBlocks生成删除数据块命令，并返回给数据节点。

####4. 数据节点删除文件数据块
数据节点在调用namenode.sendHeartbeat方法后，获得返回值DatanodeCommand数组，然后调用processCommand方法依次处理每个命令：
<pre>
private boolean processCommand(DatanodeCommand cmd) throws IOException {
    ...
    switch(cmd.getAction()) {
    	case DatanodeProtocol.DNA_TRANSFER:
      		
			...
		
		// 删除数据块命令：调用blockScanner删除blocks
    	case DatanodeProtocol.DNA_INVALIDATE:
      		Block toDelete[] = bcmd.getBlocks();
      		try {
        		if (blockScanner != null) {
          			blockScanner.deleteBlocks(toDelete);
        		}
        		data.invalidate(toDelete);
      		} catch(IOException e) {
        		checkDiskError();
        		throw e;
      		}
      		myMetrics.blocksRemoved.inc(toDelete.length);
      		break;
    	
		case DatanodeProtocol.DNA_SHUTDOWN:
	      	...
    	case DatanodeProtocol.DNA_REGISTER:
	      	...
    	case DatanodeProtocol.DNA_FINALIZE:
      		...
    	case UpgradeCommand.UC_ACTION_START_UPGRADE:
      		...
    	case DatanodeProtocol.DNA_RECOVERBLOCK:
      		...
    	default:
      		...
    }
    return true;
}
</pre>

关于删除数据块的具体实现，大家有兴趣的话可以参阅DataBlockScanner.deleteBlocks()中的相关代码，在此就不再叙述了。