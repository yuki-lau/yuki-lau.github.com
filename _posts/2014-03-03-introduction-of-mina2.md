---
layout: post
title: Apache MINA 2 简介
tags: [MINA, Java]
category: MINA
---

###摘要：
> 本文重点介绍Apache MINA 2中的基本概念，结构组织和关键API。具体的编程范例可以参阅后续的相关文章。

###参考：
* [使用 Apache MINA 2 开发网络应用](https://www.ibm.com/developerworks/cn/java/j-lo-mina2/) ：本文内容主要摘自此文，原文中还包含有一个计算器、俄罗斯方块对战范例。
* [使用 Apache MINA2 实现Web系统的消息中间件](http://www.ibm.com/developerworks/cn/web/1108_sumeng_mina2/) ：原文对网站系统架构拆分、消息中间件所在层次以及与其他组件的联系有一个简单而清晰的介绍，同时包含使用MINA2开发的一个简单的消息中间件范例，主要用到了反射进行方法调用。

###目录：
* What is MINA?
* Overview of MINA
* I/O服务
	* I/O接受器（I/O acceptor）
	* I/O连接器（I/O connector）
* I/O会话
* I/O过滤器
	* I/O过滤器链
* I/O处理器
 
###What is MINA?
[Apache MINA 2 ](https://mina.apache.org/)是一个开发高性能和高可伸缩性网络应用程序的网络应用框架，当前最新的版本是2.0.7（Update to 2014-03-04），下文中我们将简称为“MINA”。它提供了一个抽象的事件驱动的异步 API，可以使用 TCP/IP、UDP/IP、串口和虚拟机内部的管道等传输方式。Apache MINA 2 可以作为开发网络应用程序的一个良好基础。

###Overview of MINA
首先，我们来介绍一下MINA中的基本概念和网络应用架构。

使用MINA，我们可以方便的编写网络通信中的Server端和Client端，无论是哪端，MINA在整个网络应用架构中都处于如下图所示的位置。

<img src="/images/201403/1.gif" />

在MINA中，主要包含**I/O服务、I/O会话、I/O过滤器（链） 和 I/O处理器**这4个关键结构，他们的组织关系如下图所示。

<img src="/images/201403/2.gif" />

接下来，让我们来详细介绍一下这四个概念。

###I/O服务

I/O服务用来执行真正的I/O操作，以及管理I/O会话。根据所使用的数据传输方式的不同，有不同的I/O服务的实现。由于I/O服务执行的是“输入”和“输出”两种操作，实际上有两种具体的子类型：

* **I/O接受器（I/O acceptor）**：用来接受连接，一般用在**服务器**的实现中；
* **I/O连接器（I/O connector）**：用来发起连接，一般用在**客户端**的实现中。

对应在Apache MINA中的实现，<code>org.apache.mina.core.service.IoService</code>是I/O服务的接口，而继承自它的接口<code>org.apache.mina.core.service.IoAcceptor</code>和<code>org.apache.mina.core.service.IoConnector</code>则分别表示I/O接受器和I/O连接器。IoService接口提供的重要方法如表 1所示。

<table>
	<caption>表1 IoService中的重要方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>setHandler(IoHandler handler)</td>
	<td>设置I/O处理器。该I/O处理器会负责处理该I/O服务所管理的所有I/O会话产生的I/O事件。</td>
	</tr>
	<tr>
	<td>getFilterChain()</td>
	<td>获取I/O过滤器链，可以对I/O过滤器进行管理，包括添加和删除I/O过滤器。</td>
	</tr>
	<tr>
	<td>getManagedSessions()</td>
	<td>获取该I/服务所管理的I/O会话。</td>
	</tr>
	</tbody>
</table>

####1. I/O接受器
I/O接受器用来接受连接，与对等体（客户端）进行通讯，并发出相应的I/O事件交给I/O处理器来处理。使用I/O接受器的时候，只需要调用**bind**方法并指定要监听的套接字地址。当不再接受连接的时候，调用**unbind**停止监听即可。

<pre>
	// 创建IoAcceptor对象，提供IO服务，接受连接
	IoAcceptor acceptor = new NioSocketAcceptor();
	
	// 添加IoFilter
	acceptor.getFilterChain().addLast("logger", new LoggingFilter());
	acceptor.getFilterChain().addLast("codec", new ProtocolCodecFilter(
													new TextLineCodecFactory(
														Charset.forName("UTF-8"))));
	// 设置IoHandler
	acceptor.setHandler(new CalculatorHandler());
		
	// 绑定端口并启动
	acceptor.bind(new InetSocketAddress(PORT));
</pre>

####2. I/O连接器
I/O连接器用来发起连接，与对等体（服务器）进行通讯，并发出相应的I/O事件交给I/O处理器来处理。使用I/O连接器的时候，只需要调用**connect**方法连接指定的套接字地址。另外可以通过**setConnectTimeoutMillis**设置连接超时时间（毫秒数）。以下代码中，首先创建一个Java NIO的套接字连接器NioSocketConnector的实例，接着设置超时时间。再添加了I/O过滤器之后，通过connect方法连接到指定的地址和端口即可。

<pre>
	// 创建IoConnector对象，提供IO服务，发起连接
	IoConnector connector = new NioSocketConnector(); 
		
	// 设置连接超时时间（毫秒数）
	connector.setConnectTimeoutMillis(CONNECT_TIMEOUT); 
		
	// 添加IoFilter
	connector.getFilterChain().addLast("logger", new LoggingFilter()); 
	connector.getFilterChain().addLast("codec", new ProtocolCodecFilter(
													new TextLineCodecFactory(
														Charset.forName("UTF-8"))));
		
	// 指定的套接字地址
	ConnectFuture connectFuture = connector.connect(new InetSocketAddress(host, port)); 
	connectFuture.awaitUninterruptibly(); 
</pre>

在上述代码中，我们可以看到ConnectFuture这个类，利用到了**Future模式**。这里我们来简单介绍一下Future模式。
> Future模式

###I/O会话
I/O会话表示一个活动的网络连接，与所使用的传输方式无关。I/O会话可以用来存储用户自定义的与应用相关的属性。这些属性通常用来保存应用的状态信息，还可以用来在I/O过滤器和I/O处理器之间交换数据。I/O会话在作用上类似于Servlet规范中的HTTP会话。

Apache MINA中I/O会话实现的接口是<code>org.apache.mina.core.session.IoSession</code>。该接口中比较重要的方法如表 2所示。
<table>
	<caption>表 2 IoSession 中的重要方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>close(boolean immediately)</td>
	<td>关闭当前连接。如果参数 immediately为 true的话，连接会等到队列中所有的数据发送请求都完成之后才关闭；否则的话就立即关闭。</td>
	</tr>
	<tr>
	<td>getAttribute(Object key)</td>
	<td>从I/O会话中获取键为 key的用户自定义的属性。</td>
	</tr>
	<tr>
	<td>setAttribute(Object key, Object value)</td>
	<td>将键为key，值为value的用户自定义的属性存储到I/O会话中。</td>
	</tr>
	<tr>
	<td>removeAttribute(Object key)</td>
	<td>从I/O会话中删除键为key的用户自定义的属性。</td>
	</tr>
	<tr>
	<td>write(Object message)</td>
	<td>将消息对象message发送到当前连接的对等体。该方法是异步的，当消息被真正发送到对等体的时候，IoHandler.messageSent(IoSession,Object)会被调用。如果需要的话，也可以等消息真正发送出去之后再继续执行后续操作。</td>
	</tr>
	</tbody>
</table>

###I/O过滤器
从I/O服务发送过来的所有I/O事件和请求，在到达I/O处理器之前，会先由I/O过滤器链中的I/O过滤器进行处理。MINA中的过滤器与Servlet规范中的过滤器是类似的。

过滤器可以在很多情况下使用，比如记录日志、性能分析、访问控制、负载均衡和消息转换等。过滤器非常适合满足网络应用中各种横切的非功能性需求。在一个基于 Apache MINA的网络应用中，一般存在多个过滤器。这些过滤器互相串联，形成链条，称为过滤器链。每个过滤器依次对传入的I/O事件进行处理。当前过滤器完成处理之后，由过滤器链中的下一个过滤器继续处理。当前过滤器也可以不调用下一个过滤器，而提前结束，这样I/O事件就不会继续往后传递。比如负责用户认证的过滤器，如果遇到未认证的对等体发出的I/O事件，则会直接关闭连接。这可以保证这些事件不会通过此过滤器到达 I/O处理器。

MINA中I/O过滤器都实现<code>org.apache.mina.core.filterchain.IoFilter</code>接口。一般来说，不需要完整实现IOFilter接口，只需要继承MINA提供的适配器<code>org.apache.mina.core.filterchain.IoFilterAdapter</code>，并覆写所需的事件过滤方法即可，其它方法的默认实现是不做任何处理，而直接把事件转发到下一个过滤器。

> Adapter模式

IoFilter接口大致分成两类，一类是与过滤器的生命周期相关的，另外一类是用来过滤I/O事件的。

第一类方法如表 3所示，其中参数parent表示包含此过滤器的过滤器链，参数name表示过滤器的名称，参数nextFilter表示过滤器链中的下一个过滤器。
<table>
	<caption>表 3 IoFilter 中与过滤器的生命周期相关的方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>init()</td>
	<td>当过滤器第一次被添加到过滤器链中的时候，此方法被调用。用来完成过滤器的初始化工作。</td>
	</tr>
	<tr>
	<td>onPreAdd(IoFilterChain parent, String name, IoFilter.NextFilter nextFilter)</td>
	<td>当过滤器即将被添加到过滤器链中的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>onPostAdd(IoFilterChain parent, String name, IoFilter.NextFilter nextFilter)</td>
	<td>当过滤器已经被添加到过滤器链中之后，此方法被调用。</td>
	</tr>
	<tr>
	<td>onPreRemove(IoFilterChain parent, String name, IoFilter.NextFilter nextFilter)</td>
	<td>当过滤器即将被从过滤器链中删除的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>onPostRemove(IoFilterChain parent, String name, IoFilter.NextFilter nextFilter)</td>
	<td>当过滤器已经被从过滤器链中删除的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>destroy()</td>
	<td>当过滤器不再需要的时候，它将被销毁，此方法被调用。</td>
	</tr>
	</tbody>
</table>

第二类方法如 表 4 所示。
<table>
	<caption>表 4 IoFilter 中过滤I/O事件的方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>filterClose(IoFilter.NextFilter nextFilter, IoSession session)</td>
	<td>过滤对IoSession的close方法的调用。</td>
	</tr>
	<tr>
	<td>filterWrite(IoFilter.NextFilter nextFilter, IoSession session, WriteRequest writeRequest)</td>
	<td>过滤对IoSession的write方法的调用。</td>
	</tr>
	<tr>
	<td>exceptionCaught(IoFilter.NextFilter nextFilter, IoSession session, Throwable cause)</td>
	<td>过滤对IoHandler的exceptionCaught方法的调用。</td>
	</tr>
	<tr>
	<td>messageReceived(IoFilter.NextFilter nextFilter, IoSession session, Object message)</td>
	<td>过滤对IoHandler的messageReceived方法的调用。</td>
	</tr>
	<tr>
	<td>messageSent(IoFilter.NextFilter nextFilter, IoSession session, WriteRequest writeRequest)</td>
	<td>过滤对IoHandler的messageSent方法的调用。</td>
	</tr>
	<tr>
	<td>sessionClosed(IoFilter.NextFilter nextFilter, IoSession session)</td>
	<td>过滤对IoHandler的sessionClosed方法的调用。</td>
	</tr>
	<tr>
	<td>sessionCreated(IoFilter.NextFilter nextFilter, IoSession session)</td>
	<td>过滤对IoHandler的sessionCreated方法的调用。</td>
	</tr>
	<tr>
	<td>sessionIdle(IoFilter.NextFilter nextFilter, IoSession session, IdleStatus status)</td>
	<td>过滤对IoHandler的sessionIdle方法的调用。</td>
	</tr>
	<tr>
	<td>sessionOpened(IoFilter.NextFilter nextFilter, IoSession session)</td>
	<td>过滤对IoHandler的sessionOpened方法的调用。</td>
	</tr>
	</tbody>
</table>
对于表 4中给出的与I/O事件相关的方法，它们都有一个参数是nextFilter，表示过滤器链中的下一个过滤器。如果当前过滤器完成处理之后，可以通过调用nextFilter中的方法，把I/O事件传递到下一个过滤器。如果当前过滤器不调用 nextFilter中的方法的话，该I/O事件就不能继续往后传递。另外一个共同的参数是session，用来表示当前的I/O会话，可以用来发送消息给对等体。下面通过具体的实例来说明过滤器的实现。

####过滤器链
过滤器只有在添加到过滤器链中的时候才起作用。过滤器链是过滤器的容器。**过滤器链与I/O会话是一一对应的关系**。<code>org.apache.mina.core.filterchain.IoFilterChain</code>是Apache MINA中过滤器链的接口，其中提供了一系列方法对其中包含的过滤器进行操作，包括查询、添加、删除和替换等。如表 5所示。

<table>
	<caption>表 5 IoFilterChain 接口的方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>addFirst(String name, IoFilter filter)</td>
	<td>将指定名称的过滤器添加到过滤器链的开头。</td>
	</tr>
	<tr>
	<td>addLast(String name, IoFilter filter)</td>
	<td>将指定名称的过滤器添加到过滤器链的末尾。</td>
	</tr>
	<tr>
	<td>contains(String name)</td>
	<td>判断过滤器链中是否包含指定名称的过滤器。</td>
	</tr>
	<tr>
	<td>get(String name)</td>
	<td>从过滤器链中获取指定名称的过滤器。</td>
	</tr>
	<tr>
	<td>remove(String name)</td>
	<td>从过滤器链中删除指定名称的过滤器。</td>
	</tr>
	<tr>
	<td>replace(String name, IoFilter newFilter)</td>
	<td>用过滤器newFilter替换掉过滤器链中名为name的过滤器。</td>
	</tr>
	<tr>
	<td>getSession()</td>
	<td>获取与过滤器链一一对应的 I/O 会话。</td>
	</tr>
	</tbody>
</table>

###I/O处理器
I/O事件通过过滤器链之后会到达I/O处理器。I/O处理器中与I/O事件对应的方法会被调用。MINA中<code>org.apache.mina.core.service.IoHandler</code>是I/O处理器要实现的接口，一般情况下，只需要继承自 <code>org.apache.mina.core.service.IoHandlerAdapter</code>并覆写所需方法即可。IoHandler接口的方法如表 6所示。

<table>
	<caption>表 6 IoHandler 接口的方法</caption>
	<thead><tr><th>方法</th><th>说明</th></tr></thead>
	<tbody>
	<tr>
	<td>sessionCreated(IoSession session)</td>
	<td>当有新的连接建立的时候，该方法被调用。</td>
	</tr>
	<tr>
	<td>sessionOpened(IoSession session)</td>
	<td>当有新的连接打开的时候，该方法被调用。该方法在sessionCreated之后被调用。</td>
	</tr>
	<tr>
	<td>sessionClosed(IoSession session)</td>
	<td>当连接被关闭的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>sessionIdle(IoSession session, IdleStatus status)</td>
	<td>当连接变成闲置状态的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>exceptionCaught(IoSession session, Throwable cause)</td>
	<td>当I/O处理器的实现或是Apache MINA中有异常抛出的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>messageReceived(IoSession session, Object message)</td>
	<td>当接收到新的消息的时候，此方法被调用。</td>
	</tr>
	<tr>
	<td>messageSent(IoSession session, Object message)</td>
	<td>当消息被成功发送出去的时候，此方法被调用。</td>
	</tr>
	</tbody>
</table>

对于表 6中的方法，有几个需要重点的说明一下。

* sessionCreated和sessionOpened的区别：sessionCreated方法是由I/O处理线程来调用的，而sessionOpened是由其它线程来调用的。因此从性能方面考虑，**不要在sessionCreated方法中执行过多的操作**。
* sessionIdle，默认情况下，闲置时间设置是禁用的，也就是说sessionIdle并不会被调用。可以通过IoSessionConfig.setIdleTime(IdleStatus, int)来进行设置。
	

