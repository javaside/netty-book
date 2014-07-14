===========================
第六章 ChannelHandler
===========================
(*翻译很生硬，仅做互相学习交流使用，发现问题欢迎反馈。2014-07-11更新*)


本章内容

	* ChannelPipeline
	* ChannelHandlerContext
	* ChannelHandler
	* InBound和OutBound


接受和创建连接只是你应用程序的一部分，我们还需要编写很复制的代码去处理输入和输出的数据。

Netty 提供了很强大的方法来处理这些问题。他允许在处理数据的ChannelHandler上使用钩子。他还提供了ChannelHandler链，每个ChannelHander处理小的任务。这样有助于你写的干净和可重用的实现。

用ChnnelHandler处理数据也只是其中一个用法，你还可以用来废止I/O操作，而所有这一切都是实时的。


ChannelPipeline
==================

ChannelPipeline 是一个list,包含多个拦截和处理输入和输出操作的ChannelHandler实例。ChannelPipeline提供了拦截过滤器模式的一种高级形式，让用户完全控制如何处理事件，以及如何在ChannelPipeline的ChannelHandlers彼此交互。

对于每一新的channel,一个新的ChannelPipeline被创建和添加到channel中。一旦添加到channel中，channel和CahnnelPileline的关联是不变的。channel不能添加其他的ChannelPipelie,也不能分离channle当前关联的ChannelPipeline。所有这些已经帮你处理了，你不需要处理他。

下图展示了ChannelHandlers在一个ChannelPipeline中典型的处理I/O流程。一个I/O操作可以被ChannelInboundHendler和ChannelOutboundHandler其中一个处理，然后通过调用ChannelInboundInvoker或者ChannleOutboundInvoker接口定义的方法转向最近的一个handler处理。ChannelPipeline扩展了他们两个。

(*新版中ChannelInboundInvoker，ChannleOutboundInvoker已废弃。使用TailContext，HeadContext内部类替代*)

.. image:: _static/image/6.1.png


从上图可以看出，ChannelPipeline主要是ChannelHandler组成的一个列表(list)。如果一个inbound(读数据) I/O事件被触发，他会从ChannelPipeline开始传递到末端。对于outbount(写数据) I/O 事件，他会从ChannelPipelined的末端传递到开始。ChannelPipeline通过检查类型，知道ChannelHandler能否处理这事件。如果不能处理他，他会跳过这个ChannelHandler，使用下一个匹配的ChannelHandler处理。在ChannelPipeline上的修改是实时的，这意味着你可以添加、删除、替换ChannelHandler。这允许写入灵活的逻辑，如多路复用器。在本章后面我们会更详细的说明。

现在让我们看一下怎样修改一个ChannelPipeline。

.. image:: _static/image/t6.1.png

下面代码演示了如何使用这些方法来修改ChannelPipeline.

*Listing 6.1 Modify the ChannelPipeline*
::
	ChannelPipeline pipeline = ..;
	FirstHander firstHandler = new FirstHandler();			#1
	pipeline.addLast("handler1", firstHandler);			#2
	pipeline.addFirst("handler2", new SecondHandler());		#3
	pipeline.addLast("handler3", new ThirdHandler());		#4
	
	pipeline.remove("handler3");					#5
	pipeline.remove(firstHandler);					#6

	pipeline.replace("handler2","handler4", new ForthHandler());    #7


	#1 创建一个FirstHanlder实例
	#2 添加FirstHandler实例到ChannelPipeline中。
	#3 添加SecondHandler实例到ChannelPipeline第一个位置中，这意味着他在已存在的FirstHandler前面。
	#4 添加ThirdHandle到ChannelPipeline中的最后位置。
	#5 删除名称为handler3的ThirdHanlder。
	#6 通过引用实例删除FirstHandler。
	#7 用ForthHandler替换已添加的handler2，并且命名为hander4


正如你看到的，修改ChannelPipeline很容易，并且允许你根据需求添加，删除，替换ChannelHandler。


他允许你修改ChannelPipeline，也有一些让你访问被添加的ChannelHandler实现，来检查特定的ChannelHandler是否存在ChannelPipeline中。


.. image:: _static/image/t6.2.png


ChannelPipeline 继承了ChannelInboundInvoker和ChannelOutboundInvoker，他暴露了调用inbound和outbound操作的额外方法。这些操作对于通知每一个ChannelPipeline中的ChannelInboundHandler处理不同的事件。（这些方法不再一一列出，请看API docs）


(*新版中ChannelInboundInvoker，ChannleOutboundInvoker已废弃。使用TailContext，HeadContext内部类替代。*)



ChannelHandlerContext
==========================

每当一个ChannelHandler添加到ChannelPipeline，一个新的ChannelHandlerContext会被创建并且关联他。ChannelHandlerContext 允许ChannelHandler之间交互在同一个传输下，这部分和ChannelPipline有点相同。

对于一个被添加的ChannelHandler，他的ChannelHandlerContext永远不会被改变，所以他可以从缓存中安全获取。

ChannelHandlerContext实现了ChannelInboundInvoker和ChannelOutboundInvoker。他有很多方法都出现在Channel和ChannelPipeline中。不同之处是如果你调用这些方法在Channel或者ChannelPipeline上，他们总是经过完整的ChannelPipeline。相比之下，如果你调用ChannelHandlerContext中的方法，他从当前位置开始，通知在ChannelPipeline里最近的ChannelHandler来处理事件。


通知下一个ChannelHandler
----------------------------

你可以通过调用ChannelInboundInvoker和ChannelOutbountdInvoker中的方法来通知在相同ChannelPipeline中最近的handler。从哪里开始通知，取决于你对通知的设置。

下图中显示ChannelHandlerContext是如何属于ChannelHandler和绑定到ChannelPipeline上的。

.. image:: _static/image/6.2.png

现在，如果你想有贯穿整个ChannelPipeline的事件流，这里有两个不同的方法：

	* 调用Channel上的方法
	* 调用ChannelPipeline上的方法。

这两种方法让事件流贯穿整个ChannelPipeline，从头开始还是从末尾开始贯穿ChannelPipeline要取决于事件的性质，如果是一个inbound时间，他从头开始，如果是一个outbound事件，他从尾开始。

下面代码展示一个write事件怎么从ChannelPipeline的尾端开始通过ChannelPipeline。（他是一个outbound操作）

*Listing 6.2 Events via Channel*
::
	ChannelHandlerContext ctx = ..;
	Cahnnel channel = ctx.channel();						#A
	channel.write(Unpooled.copiedBuffer("Action in Action",CharsetUtil.UTF_8)	#B
	
	#A 获取属于ChannelHandlerContext的Channel引用。
	#B 通过channel写数据。


这个消息流通过整个ChannelPipeline。你可以通过ChannelPipeline做同样的事情。如下所示

*Listing 6.3 Events via ChannelPipeline*
::
	ChannelHandlerContext ctx = ..;
	ChannelPipeline pipeline = ctx.pipeline();					#A
	pipeline.write(Unpooled.copiedBuffer("Action in Action",CharsetUtil.UTF_8)	#B

	#A 从ChannelHandlerContext 获取一个ChannlePipeline的引用
	#B 通过ChannelPipeline 写数据。


消息流通过整个ChannelPipeline。上面的两个列子在事件流关系上，她们的操作都是相等的。你也应该注意到Channel和ChannelPipeline都可以从ChannelHandlerContext访问到。

下图展示被Channel或者ChannelPipeline触发的事件流。

.. image:: _static/image/6.3.png




