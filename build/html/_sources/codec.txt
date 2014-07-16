============================
第七章 Codec
============================
(*翻译很生硬，基本不通，不要浪费时间。本人也是读这本书时顺便记录下来。2014-07-16更新*)

本章覆盖内容
	* Codec
	* Decoder
	* Encoder

在上一章，你学习了各种各样的方法处理处理链的钩子和如何拦截操作或者数据。虽然ChannelHandler能够实现各种类型的逻辑，但是仍然有改进的余地。

为了满足这，Netty 提供了Codec 框架，他使得实现你的协议非常的简单，并且很容易重用和包装。

这章我们讨论code框架的不同部分以及如何使用他。



Codec(编码解码器)
====================

每当你写基于网络的程序时，你都需要实现各种编码解码器。编码解码器规定把原始的字节解析和转换成各种类型的自定义的逻辑消息。同样的也需要把消息转换成原始字节，可以在网络上传输。

记住，在网络上传输数据都是使用字节传输。一个编码解码器有两部分组成：
	* Decoder(解码器)
	* Encoder(编码器)

decoder是负责把字节解码成为消息。encoder正好相反，他负责把消息解码成字节。

我们应该清楚decoder是为inbound 处理数据，encoder是为outbound处理数据。

让我们看看decoder和encoder的各种抽象类。

Decoders(解码器)
=====================

Netty提供了很丰富的抽象类帮助你很容易的实现编写解码器。这里分为不同的类型。

	* 把字节解码成消息
	* 把消息解码成消息
	* 办消息解码成字节

本节将概述不同的抽象类，你可以使用他们来实现你的解码器，帮助你理解什么是好的解码器。

在我们投入Netty提供的实际抽象类之前，让我们定义一下解码器的职责。解码器负责把inbound数据从一种格式解码成另一种格式。因为解码器处理inbound数据，所以他是一个ChannelInboundHandler抽象类的实现。

你可以问你自己，在实践中你什么时候需要一个解码器。这非常简单，每当你在ChannelPipeline里需要传输inbound数据到下一个ChannelInboundHandler的时候。


甚至比你想象中更灵活，你可以按需要尽可能多的把解码器放到ChannelPipeline中，考虑ChannelPipeline的设计，他允许你组装可重用的组件逻辑。


ByteToMessageDecoder
------------------------

你经常想把字节解码成消息，甚至把字节解码成其他顺序的字节。这是很常见的任务，Netty已经抽象了基本的类去使用他。

这种情况你可以使用ByteToMessageDecoder。他是一个抽象的基类，让你很容易的编写解码器实现把字节解码成对象。

ByteToMessageDecoder提供了两个方法：

	* decode(): 这个抽象方法，你必须实现，他在接收所有的字节数据是被调用，并且把解码的消息加入到列表中(list)。解码一些东西时，这方法会被调用。
	* decodeLast(): 当Channel关闭时，这方法只会被调用一次，如果在这里你需要做特别处理，你可以实现这个方法。

设想一下，我们有一个从远端写入的字节流，他包含一下简单的整数，在ChannelPipeline里，我们想后面分别的处理整数，所以我们想从inbound读取整数，然后把每个整数转发到下一个ChannelInboundHandler处理。

.. image:: _static/image/7.2.png

从上图可以看出，ToIntegerDecoder从inbound ByteBuf读取数据，然后把她们解码，并且把解码后的消息(整数)转发到下一个ChannelInboundHandler。上图也显示了每个整数需要ByteBuf里的四个字节。

下面代码演示如何实现这些。

*Listing 7.3 ByteToMessageDecoder that decodes to Integer*
::
	public class ToIntegerDecoder extends ByteToMessageDecoder{							#1
		
		@Overide
		public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception{
			if(in.readableBytes() >= 4){									#2
				out.add(in.readInt());									#3		
			}
		}
	}

	#1 继承ByteToMessageDecode 实现把字节转换成消息
	#2 检查是否至少有四个字节
	#3 从inbound ByteBuf读取整数，并加入到解码消息列表中。

使用ByteToMessageDecoder使得编写从字节到消息的解码器非常容易。但是我们可能注意到令人讨厌的小事情。在你读取实际内容之前，你需要检查出入的ByteBud是否有足够的可读字节。

如果这是不必要的岂不更好?是的，这是可以的，下节我梦将讲到。

更多复杂的例子，请参考LineBasedFrameDecoder。这是Netty自己的一部分，你可以在io.netty.handler.code包里找到。此外还包含了很多其他的实现，把字节专卖成消息是经常用到的。

ReplayDecoder
-----------------
