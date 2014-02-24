---
layout: post
title: Hadoop源码阅读、开发环境（win7 + cygwin + eclipse + hadoop 0.20.2）
tags: [Hadoop, HDFS]
category: HDFS
---

###摘要：

> 本文记录了在Windows环境下，使用Cygwin模拟Unix环境，部署Hadoop的基本方式。并将Hadoop源码导入Eclipse工程，从而方便对Hadoop源码的学习、简单的二次开发和验证。

虽然在真实的生产环境中，Hadoop都是以大规模集群的形式存在的，但在初始学习的过程中，我们可以先在本机上搭建一个单机的Hadoop环境，并将源码导入Eclipse进行学习。最好的方式当然还是将Hadoop部署在Linux上，但如果是Windows环境，我们也可以安装cygwin来模拟Unix环境，实现在Windows上学习Hadoop的基本操作、使用、源码和二次开发。

###目录：

* 环境准备
* 源码准备
	* 下载Hadoop源码
	* 下载配置Ant
	* 将源码导入Eclipse
* 编译Eclipse源码
	* 使用Ant Build源码
	* 验证

###一、环境准备
环境准备包括三个方面：

* 安装JDK
* 安装Cygwin
	* 配置环境变量
	* 安装sshd服务
	* 启动sshd服务
	* 配置ssh登录
* 部署Hadoop到Cygwin
	* 下载Hadoop安装包
	* 安装Hadoop
	* 启动Hadoop

以上过程，网上的资料很多，例如：

* <a href="http://wenku.baidu.com/view/6af47921af45b307e8719799.html" target="_blank">在Windows上安装Hadoop教程</a>
* Hadoop官方的<a href="http://v-lad.org/Tutorials/Hadoop/00%20-%20Intro.html" target="_blank">Hadoop on Windows With Eclipse</a>：官方版本总是好的，虽然因为版本的原因，部分内容有所区别，但可以尽情搜索解决。

在进行Cygwin的安装时，请注意以下几点：

* 请到<a href="http://cygwin.com/install.html" target="_blank">cygwin官网下载页</a>，下载 <b>最新版本</b> 的setup.exe，否则在后续通过internet继续安装其他依赖包时会出错。
* 在选择镜像站点时，请选择 http://mirrors.163.com。如果多个站点均出现“Unable to get setup.ini from <xxx>”这样的错误，请仔细确认当前的cygwin是否为最新版本。
* 在下载过程中，如果提前终止，出现“incomplete”错误，可以多retry几次。
* 我在这里使用的是hadoop-0.20.2版本：<a href="https://archive.apache.org/dist/hadoop/core/hadoop-0.20.2/hadoop-0.20.2.tar.gz" target="_blank">https://archive.apache.org/dist/hadoop/core/hadoop-0.20.2/hadoop-0.20.2.tar.gz</a>
* 在hadoop-env.sh设置JAVA_HOME时，如果你的路径上有空格，那就悲剧了，需要没空格，重新安装吧。然后，按照如下规则进行设置。

<pre>
	#JAVA_HOME的值
	C:\Java\jdk1.7.0_45
	#hadoop-env.sh应当设置的值
	export JAVA_HOME=/cygdrive/C/Java/jdk1.7.0_45
</pre>

###二、源码准备

####1. 下载Hadoop源码
Hadoop官方提供了SVN方式的代码下载，在这里下载0.20.2版本。SVN地址为:
<code>http://svn.apache.org/repos/asf/hadoop/common/tags/release-0.20.2</code>
例如，下载到 D:\hadoop0.20.2

####2. 下载配置Ant
到Apache Ant官网下载Ant bin文件，例如当前最新版本为 <a href="http://mirrors.hust.edu.cn/apache//ant/binaries/apache-ant-1.9.3-bin.zip" target="_blank">1.9.3</a>，下载完毕后，解压即可，如解压到D:\apache-ant-1.9.3。
新建环境变量 ANT\_HOME 值为 <code>D:\apache-ant-1.9.3</code>，将 <code>%ANT_HOME%\bin</code> 添加到环境变量Path中。

####3. 将源码导入Eclipse
#####1) Ant Eclipse-files
我们checkout下来的hadoop源码根目录中，有一个ant配置文件：build.xml，它提供了eclipse任务，该任务可以为Hadoop代码生成Eclipse项目文件，免去创建Eclipse项目所需的大量配置工作。
我们在命令行下，进入源码根目录 D:\hadoop0.20.2，执行命令：
<pre>
	ant eclipse-files
</pre>
等一会儿，命令执行完毕（如下图所示），就可以准备将源码导入eclipse了。

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-1.png" />

#####2) 建立Eclipse项目
打开Eclipse，选择File -> New -> Java Project，勾掉“Use Default Location”，选择项目的位置为Hadoop的根目录，即 D:\hadoop0.20.2，然后单击“Finish”按钮，就完成了Eclipse项目的创建。

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-2.png" />

#####3) 添加ANT_HOME变量
完成以上步骤后，可以发现eclipse报告了一个错误“Unbound classpath variable: 'ANT\_HOME/lib/ant.jar in project xxx'”。显然我们需要设置eclipse的ANT\_HOME变量。
右键hadoop项目 -> Properties -> Java Build Path，在Libraries页面编辑出错的项：ANT\_HOME/lib/ant.jar，创建变量ANT_HOME，其值为Ant的安装目录。

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-3.png" />

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-4.png" />

#####4) Source选择（可选）
由于我个人比较关注的是HDFS模块，当然也会涉及到Common模块，因此我在Properties -> Java Build Path的Source页只保留了两个目录，分别是core和hdfs。

完成以上步骤后，Hadoop源码的Eclipse项目就建立完成了。

###三、编译Eclipse源码
####1. 使用Ant Build源码
#####1) 处理依赖lib缺失
如果这个时候，eclipse还是报hadoop项目中有一些lib缺失，不要紧张，右键 build.xml -> run as -> ant build。在build的过程中会将依赖的lib进行下载的。
如果还是不行，先不管这一步，继续创建Ant Builder。

#####2) 创建Ant Builder
右键hadoop项目 -> Properties -> Builders -> New -> Ant Builder。

1. 输入Builder的名称：Hadoop_Builder
2. 在main标签页下选择 Browse File System，选中hadoop源码根目录下的builder.xml
3. 选中targets标签页，在Manual Build部分选择Set Targets，去掉default勾选，选择jar，点击OK完成

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-5.png" />

1. 在properties -> builders页面，如果在这里已经存在了一个Hadoop_Ant_Builder了的话，删掉它，以我们刚刚创建的为准。
2. 选中Java Builder，点击Down，同时去掉勾选Java Builder。选择OK将Hadoop_Builder移至最上。
3. 选中eclipse菜单栏的Project，在Project菜单中去掉Build Automatically前面的勾，选择Build Project编译工程。

如果编译失败了，不妨多试几次，就能看到如下结果了。

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-6.png" />

####2. 验证
现在我们来验证build出的jar是否可以很好的完成hadoop原本的功能。
我在DataNode类中的main方法，添加了一条INFO日志，如下所示：

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-7.png" />

选择 Project -> Build Project编译工程，成功后，至D:\hadoop-0.20.2\build目录下，将生成的hadoop-0.20.3-dev-core.jar复制到cygwin中的hadoop根目录下。
备份原有的hadoop-0.20.2-core.jar，将hadoop-0.20.3-dev-core.jar重命名为hadoop-0.20.2-core.jar，执行start-dfs.sh命令启动HDFS。
启动完成后，我们可以在DataNode的输出日志中看到如下信息，表明我们所添加的修改成功应用到Hadoop上啦。

<img src="/images/201312/installing-hadoop-on-windows-with-cygwin-and-developing-with-eclipse-8.png" />

<hr />

至此，在Windows环境下部署Hadoop，并使用Eclipse阅读、二次开发源码的过程就全部介绍完了。虽然在Windows下进行Hadoop学习有点“不伦不类”，但我自己还是希望在学习的初期不要被陌生的环境所打消信心，待后续更加熟悉Hadoop（其实主要目标是HDFS^ ^）后，再进行Linux迁移。

网上关于以上过程介绍的资料也很多，我仅记录下自己验证有效的，也方便给实验室后续的师弟师妹们做快速入门啦，也算是给自己多日折腾的结果做一个总结。