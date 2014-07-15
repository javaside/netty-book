======================
第五章 Buffers
======================

(*翻译很生硬，基本不通，不要浪费时间。本人也是读这本书时顺便记录下来。2014-07-09更新*)





Buffer API
=============


Netty的Buffer API 包含两个接口:

* ByteBuf

* ByteBufHolder

Netty使用“引用计数”的方法来知道什么时候安全释放一个Buf及其声明的资源。理解Netty使用“引用计数”这是非常有用的，他的所有处理都是自动的。
Netty使用缓存池和其他技巧来加速并且保持内存使用率合理的水平。对于上述你不需要去做任何事情，但是当你去开发一个Netty的应用，你应该尽快去处理你的数据并且释放缓存池的资源。

Netty的Buffer API 提供几个优势:

* 可以定义自己的Buffer类型。

* 透明的零拷贝的实现是通过内置的复合Buffer类型构建的。

* 容量可以按需要扩大，类似StringBuffer。

* 切换读写模型不需要访问flip()方法。

* 读写索引是分开的。

* 方法链式调用。
  
* 引用计数

* 缓存池

在本章的最后一节我们深入来探讨一下这些优势，包括缓存池。让我们从字节容器开始，这个可能是第一个需要的。

ByteBuf-字节容器
==================

每当你需要远端交互时，比如数据库，其相互通讯都是需要用字节来完成的。对于这点和其他原因来说，一个高效，方便和有用的数据结构是必须的，Netty的ByteBuf实现和满足这些要求，做成一个最合理的数据容器，优化用字节处理和交互。

ByteBuf是一个高效读写字节的数据容器。这样的读写操作非常简单，他提供了读写两个索引。允许你按顺序读取数据和跳回重新读取数据。你只需要调整读索引然后开始重新读操作。


怎样工作的
------------

在一些数据写入到ByteBuf，他的writeIndex根据写入的数量自动增长。在你开始读取数据之后，他的readerIndex索引也会自己增长。你可以读取从readerIndex到writerIndex之间的字节。当readerIndex和writerIndex位置相同时,ByteBuf是不可以读取的，所以继续读取会引发IndexOutOfBoundsException异常，类似读取超过数组容量的现象。

访问Buffer的任何方法开始读写，读写索引都会自动的移动。还有些相关的操作比如“set”和“get”字节也是一样的会自动更新对应的读写索引。这些都不需要去自己再去移动相关的索引，但是操作相关的索引是可以的。

一个ByteBuf有一个最大的容量来设置可以处理的最大的数据的上限，试图移动读写索引超过最大容量将会获得一个异常。默认最大限制是Integer.MAX_VALUE.

.. image:: _static/image/5.1.png

从图5.1可以看出，ByteBuf类似一个字节数组，最大的不同就是增加了能控制访问buffer数据的读写索引。

在后面的章节，你将学习到更多ByteBuf可执行的操作。现在，让我们来回顾一下你最可能用到不同类型的ByteBuf。


ByteBuf 不同类型
------------------

当你用Netty时，你会遇到三种不同的ByteBuf类型(有更多的类型，但是都是内部使用)。你可以终实现自己的ByteBuf，但是这已经超出我们讨论的范围。让我们看看，你最有可能感兴趣提供的类型。


HEAP BUFFERS
______________

最常用到的ByteBuf类型是存储在JVM的堆内存中。这是通过将存储在一个底层数组实现的。当你没有用到缓存池，这种类型是很快的分配和销毁的。他当然也提供直接访问底层数组，这使得很容易的与原来编写的代码整合。访问不是堆ByteBuf的数组将会抛出UnsupportedOperationException异常。正因如此，所以最好每次都用hassArray()方法检查是不是底层数组。如果你曾今用过JDK的ByteBuffer你会觉得这种模式非常熟悉。


DIRECT BUFFERS
_______________

其他的ByteBuf的实现都是Direct Buffer。Direct的意思就是在分配内存时直接从JVM堆内存以外的主存中分配。在堆内存中你看不见内存的使用情况。你必须考虑计算你的应用将会用到的最大内存以及如何限制的使用他，只考虑最大堆大小是不够的。当在socket传输数据的时候，使用Direct Buffer是最佳的选择。事实上，如果你使用非Direct buffer，JVM内部将从你的Buffer中拷贝到Direct Buffer，然后再在socket中传输。

Direct Buffer在分配和销毁方面要比heap buffer昂贵的多。这也是为什么Netty支持缓存池来消除这些问题的一个原因。你不能通过底层数组来访问这些数据，所以，如果你需要兼容原来的代码，你必须拷贝这些数据。

下面将展示，在没有访问底层数组的情况下，你如何获取数组中的数据并且调用你的方法。

.. image:: _static/image/5.2.png 

正如你看到的，Direct Buffer 需要更多的工作，涉及到复制操作。如果你希望访问数据是一个数组，你最好使用heap buffer。

COMPOSITE BUFFERS
__________________

最后一个ByteBuf的实现是CompositeByteBuf。从他的名字就知道是什么意思，他可以组合不同的ByteBuf实例，并且提供基于其不同实例之上的一个视图。你还可以即时的增加和删除这些ByteBuf实例，就像一个List一样。如果你以前使用JDK的ByteBuffer，你会发现你错过了这些功能。CompositeByteBuf只是一个基于其他ByteBuf之上的视图，这个hasArray()方法将会返回false,因为他包含了几个Direct 和 非Direct类型的ByteBuf实例。

例如，一条消息由heade和body两部分组成。在模块的应用程序中，当消息发送出去后，这两部分可能被不同的模块生成和组装。你也可以一直使用同一个body,而仅仅改变header。所以，这也让我体会到，这里任何时间都没有分配一个新的的buffer。

CompositeByteBuf非常完美，他不需要内存拷贝，而且提供非组合buffer相同的API。

下图展示一个CompositeByteBuf如何用来组成header和body。

.. image:: _static/image/5.2-1.png 

如果你使用JDK的ByteBuffer,这些是做不到的。他仅仅有一个方法去组合两个ByteBuffers，就是拷贝他们两个ByteBuffer中的内容到一个新创建的数组或者ByteBuffer中。
下面代码展示了这些操作原理。

*Listing 5.3 Compose legacy JDK ByteBuffer*
::

	// Use an array to composite them
	ByteBuffer[] message = new ByteBuffer[] { header, body };
	// Use copy to merge both
	ByteBuffer message2 = ByteBuffer.allocate(
	￼
	message2.put(header);
	message2.put(body);
	message2.flip()

从上面代码我们可以看出有几个不利条件：如果支持两个Buffer操作，不能保持原来简单的API。使用内存拷贝来完成关联两个buffer是有性能消耗的。这样很简单，但不是最佳的处理方式。

让我们看看CompositeByteBuf的使用。如下所示：

*Listing 5.4 CompositeByteBuf in action*
::

	CompositeByteBuf compBuf = ...;
	ByteBuf heapBuf = ...;
   	ByteBuf directBuf = ...;
	compBuf.addComponent(heapBuf, directBuf);       #1
	.....
	compBuf.removeComponent(0);                     #2
	for (ByteBuf buf: compBuf) {                    #3 
		System.out.println(buf.toString());
	}
	
	#1 Append ByteBuf instances to the composite (追加ByteBuf实例到组合Buffer)
	#2 Remove ByteBuf on index 0 (heapBuf here) (移除第一个ByteBuf)
	#3 Loop over all the composed ByteBuf (循环所有的组合ByteBuf)

CompositeByteBuf里面还有更多的方法，但是我认为你会很容易看明白是什么意思。Netty的API都有很清楚的描述，以便你使用未演示的其他方法。参照API文档很容易的理解他们是做什么的。

因为CompositeBytebuf的性质，你将无法访问底层实现数组。

*Listing 5.5 Access data*
::
	CompositeBuf compBuf = ...;
   	if (!compBuf.hasArray()) {                                 #1
	    int length = compBuf.readableBytes();                  #2 
	    byte[] array = new byte[length];                       #3 
	    compBuf.getBytes(array);                               #4 
	    YourImpl.method(array, 0, array.length);               #5
	}

	#1 Check if ByteBuf not backed by array which will be false for a composite buffer (检查不是底层数组实现)
	#2 Get amount of readable bytes （获取可读的字节数）
	#3 Allocate new array with length of readable bytes (分配可读字节数长度的数组)
	#4 Read bytes into array （读取字节到数组中）
	#5 Call method that takes array, offset, length as parameter(访问数组)

CompositeByteBuf是ByteBuf的一个子类，你可以像正常的ByteBuf操作,他还提供其他额外的操作。

Netty使用CompositeByteBuf会尽可能的优化在socket上的读写操作。这就意味着Netty使用聚散读写socket是不会有性能的损失，或者从JDK内存泄漏问题受到影响。所有的这一切在Netty的核心里已经做了，所以你不用担心太多。


CompositeByteBuf类并不存在JDK中，他只是提供了比JDK java.nio包中的Buffer API 更丰富的功能。

ByteBuf的字节操作
===================

ByteBuf提供了很多操作，他允许修改读取内容。你将很快的掌握他，他提供了更好的用户体验和性能。


随机访问索引
----------------

像一个普通的原始字节数组，ByteBuf使用从零开始的索引。这就意味着第一个字节的索引总是0，最后一个索引总是(容量-1)。可以迭代读取Buffer所有的字节(如下代码所示)，不需要知道其内部如何实现。

*Listing 5.7 Access data*
::
	ByteBuf buffer = ...;
	for (int i = 0; i < buffer.capacity(); i ++) {
        	byte b = buffer.getByte(i);
		System.out.println((char) b); 
	}

从上面代码我们可以知道，读取数据时并没有使用readerIndex和writerIndex索引。如果你需要，你可以通过调用readerIndex和writeIndex索引来访问其内部数据。


顺序访问索引
----------------

ByteBuf提供两个指针变量来实现顺序读写操作，readerIndex用于读操作，writeIndex用于写操作。这里再次和JDK的ByteBuffer不同，JDK的ByteBuffer只有一个指针，所以需要flip()方法来切换读写模式。下图显示了被两个指针分成三个区域片段。

.. image:: _static/image/5.3.png


可废弃的字节
-------------

可废弃的字节片段是已经读取过的，所以他是可丢弃的。刚开始，可废弃片段大小为0，但是他的大小随着读操作增加。这里只包含“read”操作，“get”操作是不会移动readerIndex索引。读过的片段可以通过调用discardReadBytes()方法来清除未用空间。

下图是在调用discardReadBytes()方法之前的 ByteBuf片段。

.. image:: _static/image/5.3.png


正如你看到的，可废弃片段区域包含了一些空间，他可以重新使用。我们可以通过调用discardReadBytes()方法来重新使用这些空间。

下图是调用discardReadBytes()后的ByteBuf片段。

.. image:: _static/image/5.5.png

要注意在调用discardReadBytes()之后，有没有保证可写字节内容。在大多情况下，可写的字节不会被移动，根据不同的buffer实现，甚至可以填充完成不同的数据。

当然，你可以平凡调用discardReadBytes()方法提供更多可写的空间。请注意，调用discardReadBytes()方法有可能涉及一个内存拷贝，因为它需要将可读的字节（内容）移到ByteBuf开始。这些操作是影响性能的，所以当你需要他或者从中获益时才使用他。你也需要尽快的释放内存。


可读字节(实际的内容)
-------------------------


这个部分才是真正的数据存储片段。任何以read或者skip开头的方法都是从当前的readerIndex索引开始读取或者跳过数据，并且readerIndex索引根据读取的字节数自动增长。如果read操作的参数是一个ByteBuf，这个参数Buffer的writeIndex索引也会自动增加。


如果接收参数ByteBuf没有剩余足够的内容空间，indexOtOfBoundException异常将会被触发。新分配的，包装的或者拷贝的Buffer的readerIndex默认值是0。

下面代码演示如何读取所有可读的数据。

*Listing 5.8 Read data*
::
	// Iterates the readable bytes of a buffer. 
	ByteBuf buffer = ...;
	while (buffer.readable()) {
	     System.out.println(buffer.readByte()); 
	}

可写字节
------------

这部分是未定义的空间，是用来填写数据的。任何以write开头的方法的操作都会从当前的writerIndex索引开始写入数据，并且writeIndex索引根据写入的字节数自动增加。如果write操作的的参数也是一个ByteBuf，参数ByteBuf的readerIndex也会自动增加。

如果没有剩余足够的可写字节，则会抛出IndexOutOfBoundException异常。新分配buffer的wrterIndex索引值是0。

下面代码例子演示了用随机int数字填入buffer中，直到填满为止。

*Listing 5.9 Write data*
::
	// Fills the writable bytes of a buffer with random integers. 
	ByteBuf buffer = ...;
	while (buffer.writableBytes() >= 4) {
	     buffer.writeInt(random.nextInt()); 
	}

清空buffer索引
----------------

通过调用clear()方法可以把readerIndex和writerIndex设置为0。他不会清空buffer中的内容，只是设置了两个指针指向0。请注意这里的clear的意思不同于JDK中ByteBuffer.clear()。

让我们看一下他的功能。下图是一个ByteBuf的三个不同数据片段。

.. image::  _static/image/5.6.png

上图所示，在调用clear()方法之前，有三个数据片段。在调用之后，数据片段已经被改变，如下图所示。

.. image::  _static/image/5.7.png

对比discardReadBytes()，clear()操作是非常廉价的，他只是移动指针不需要任何的内存拷贝。


查询操作
-------------

各种个样的indexOf()方法，能够帮助我们找到满足一定条件的索引。复杂的动态顺序搜索和简单的静态单字节搜索都可以使用ByteBufProcessor的实现来完成。

如果你正在解码可变长度的数据，如NULL结尾的字符串，你会发现bytesBefore(byte)方法很有用。让我们来想象一下你写的一个应用程序，需要用到socket，并且用到了NULL结尾的字符串内容。使用bytesBefore()方法，你很容易的消费这些数据，也不需要手工的去读取每个字节去检查NULL结尾的字节。不用ByteBufProcessor的话，你需要自己来实现这些所有的方法。此外，ByteBufProcessor是更有效的，因为他在处理过程中只需要很少的“约束检查”。


标记和重置
-------------

在说明之前，我们先在buffer中设置两个标记索引。其中一个是存储readerIndex索引，另一个是存储writerIndex索引。你可以调用reset方法来复位标记的读写索引(readerIndex,wrtiterIndex)。这个方法类似在InputStream中流行的mark,reset方法，但是这里的reset方法没有读限制。


此外，你也可用通过调用readerIndex(int)和writerIndex(int)方法来设置读写索引(readerIndex,writerIndex)。如果设置readerIndex或者writerIndex是一个非法的位置将会引起IndexOutOfBoundException异常。

衍生的Buffers
-----------------

根据已存在的buffer通过调用duplicte(),slice(),slice(int,int),readOnly()或者order(ByeOrder)创建一个视图。一个衍生出来的buffer有独立的readerInder,writerIndex和标记索引，但是他共享其他的内部数据，和NIO的ByteBuffer方式一样。因为共享了内部数据，他的创建是很廉价的，也是创建的优先选择方式，比如在一个操作中，你需要ByteBuf的一部分数据。

如果你需要从一个已存在的Buffer中的拷贝一个新的Buffer，可以使用copy()或者copy(int,int)方法代替。下面代码展示如何使用ByteBuf的slice方法。

*Listing 5.10 Slice a ByteBuf*
::
	Charset utf8 = Charset.forName("UTF-8");
	ByteBuf buf  = Unpooled.copiedBuffer("Netty in Action rocks!",utf8);    #1

        ByteBuf sliced = buf.slice(0,14);                                       #2
        System.out.println(sliced.toString(utf8);                               #3

        buf.setByte(0, (byte) 'J');                                             #4
        assert buf.get(0) == sliced.get(0);                                     #5

	#1 用给定的字符串创建一个ByteBuf
	#2 用buf的0～14字节切割一个新的ByteBuf
	#3 包含"Netty in Action"
	#4 更新buf中0位的字节。
	#5 比较buf和sliced的第一个字节，他们是相等的，因为她们共用同样的内容。


现在让我们来看看如何通过ByteBuf的copy来创建一个ByteBuf和slice方法创建有何不同。下面代码展示ByteBuf的copy是如何工作的。

*Listing 5.11 Copying a ByteBuf*
::
	Charset utf8 = Charset.forName("UTF-8");
	ByteBuf buf  = Unpooled.copieBuffer("Netty in Action rocks!", utf8);   #1
	
	ByteBuf copy = buf.copy(0, 14);                                        #2
	System.out.println(copy.toString(utf8);                                #3

	buf.setByte(0, (byte) 'J');					       #4
	assert buf.get(0) != copy.get(0);				       #5


	#1 用给定的字符串创建一个ByteBuf
	#2 用buf的0～14字节拷贝一个新的ByteBuf
	#3 包含"Netty in Action"
	#4 更新buf中0位的字节。
	#5 比较buf和sliced的第一个字节，他们是不相等的，因为她们没有共用同样的内容。

该API是相同的，但是修改内容的影响，衍生的ByteBuf是不相同的。

无论什么时候尽可能的使用slice方法，并且在必须的时候才使用copy方法。创建一个ByteBuf的拷贝更昂贵，因为他需要内存拷贝。


读写操作
------------

有两种类型的读写操作：

	* 基于位置索引下标读写操作(get/set)，他只读取指定位置的字节或者设置指定位置的字节。
	* 任何的读写操作(read/write)都会从当前的读写索引(readerIndex,writerIndex)开始读写，并且读写索引会自动增加。

*(译者注：简单说读写分两种情况，一种是不会改变读写索引的，另一中是会改变读写索引的。后面列出的都是一些方法，这里就不一一列出了，更多请看Netty API docs)*


其他有用的一些操作
--------------------

其实还有其他很有用的方法没有提及到，但是我们可能经常会用到。下图列出一些方法并且解释她们是做什么用的。

.. image::  _static/image/table5.5.png


你可能经常用到一下POJO对象，需要存储她们并且恢复成POJO对象。通常，保持这些被包含的对象的顺序是很重要的。为了这个目的，Netty提供了其他的数据容器叫做MessageBuf。


ByteBufHolder
----------------

通常，你有一个对象持有字节和其他的属性，比如一个HTTP的response对象。他有一些属性，如状态码，cookies等等，但是实际上他的内容是字节。


由于这种场景是常见的，Netty 提供了一个ByteBufHolder抽象类。这样使得Netty可以使用像缓存池这些高级功能，ByteBuf持有的数据可从缓存池中获取，同时Netty还可以自动释放他们。

事实上ByteBufHolder只有很少的方法去访问他持有的数据和使用引用计数。如果你想实现一个"message object" 用ByteBuf 来存储"payload/data"，最好的方法就是使用ByteBufHolder 类。

(*这接口的data()方法已经修改成content()*)


Netty buffer 的工具类
========================

使用JDK NIO API是很困难的。但是Netty各种各样的Buf 实现是非常容易使用，Netty已经提供一系列的工具类来创建和使用各种各样的Buffer。在这段，我们看三个工具类，因为你在使用Netty时最有用的工具类。


ByteBufAllocator - 分配ByteBuf
--------------------------------
前面我们提及过，Netty为各种各样的ByteBuf实现提供缓存池支持。Netty 提供了ByteBufAllocator来实现这些功能。从类的名称就可以看出他是负责分配前面提到过的各种类型的ByteBuf实例。不管是否是缓存的还是不指定的其他实现，都不会改变ByteBuf的操作。

ByteBufAllocator 提供了很多方法，这里不在一一列出，详情请看API docs。


ByteBufAllocator 里的方法提供了一些额外的参数用来设置ByteBuf的初始容量和最大容量。你要记住ByteBuf是可以根据需要扩大容量的，直到扩大到最大容量为止。


获取ByteBufAllocator的引用是很简单的，你可以通过channel或者ChannelHandlerContext获取。

Netty 有两个不同的ByteBufAllocator实现。一个是实现ByteBuf缓存池,他可以减少内存的消耗和保持最小的内存碎片。事实上这些实现已经不是本书的讨论范围，但是让我们注意他是基于jemalloc和利用操作系统相同的算法来高效的分配内存。

另外一个实现不是缓存池的。Netty 默认使用PooledByteBufAllocator，也可以非常容易的通过ChannelConfig来修改。


Unpooled - 创建Buffer很容易
------------------------------

很多场景你不能访问前面提到的ByteBuf，因为你没有ByteBufAllocator的引用。针对这个，Netty提供了一个Unpooled的工具类。这类提供了静态的帮助方法去创建没有使用缓存的ByteBuf实例。

他提供的方法不一一列举，请看API docs。

Unpooled工具类使得在Netty以外也可以很容易的使用Netty buffer API，你会发现他很有用，可以受益于高性能可扩展的Buffer API，又不需要Netty的其他部分。


ByteBufUtil - 小但是很有用
------------------------------

其他有用的类是ByteBufUtil。这个类提供一些静态的帮助方法，当我们操作ByteBuf，这些方法是很有用的。最主要的一个原因是，这些方法不依赖ByteBuf是否被缓存。


最可贵的是他提供了静态方法hexdump()。他可以输出16进制内容。在很多场景中这是很有用的。十六进制值可以很容易地转换回实际字节表示。你可能想知道为什么不能直接打印这些字节，问题是，这可能会导致难以阅读。

(完)
