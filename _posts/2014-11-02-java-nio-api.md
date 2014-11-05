---
layout: post
title: JAVA NIO 核心组件 API
category: 并发编程
---
[参考：Java NIO 系列教程  ](http://www.iteye.com/magazines/132-Java-NIO)

JAVA NIO核心组件 为Channels、Buffers、Selectors，下面分别介绍主要API及使用方式，会在接下来的几篇博客写NIO示例及分析经典应用示例。

####1.	 Channel 通道

基本上，所有的 IO 在NIO 中都从一个Channel 开始。Channel 有点象流，数据可以从Channel读到Buffer中，也可以从Buffer 写到Channel中，
但流的读写通常是单向的。通道可以异步地读写，通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。
Channel和Buffer有好几种类型。下面是JAVA NIO中的一些主要Channel的实现，这些通道涵盖了UDP 和 TCP 网络IO，以及文件IO。 

>> * FileChannel  从文件中读写数据
* DatagramChannel 能通过UDP读写网络中的数据。
* SocketChannel 能通过TCP读写网络中的数据。
* ServerSocketChannel 可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。

代码示例1，把文件中字符全部打印输出
{% highlight java %}
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");  
FileChannel inChannel = aFile.getChannel();  
ByteBuffer buf = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buf);  
while (bytesRead != -1) {  
	System.out.println("Read " + bytesRead);  
	buf.flip();
	while(buf.hasRemaining()){  
		System.out.print((char) buf.get());  
	}  
	buf.clear();  
	bytesRead = inChannel.read(buf);  
}  
aFile.close();  
{% endhighlight %}
***注意 buf.flip() 的调用，首先读取数据到Buffer，然后反转Buffer,接着再从Buffer中读取数据***

####2.	 Buffer 缓存

以下是Java NIO里关键的Buffer实现，这些Buffer覆盖了你能通过IO发送的基本数据类型：byte, short, int, long, float, double 和 char

>> * ByteBuffer
*	CharBuffer
*	DoubleBuffer
*	FloatBuffer
*	IntBuffer
*	LongBuffer
*	ShortBuffer


Buffer读写数据注意事项，参考示例代码1：

>> *	写入数据到Buffer，buffer会记录下写了多少数据。
*	读数据前，需要调用flip()方法将Buffer从写模式切换到读模式
*	从Buffer中读取数据，一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入
*	调用clear()或compact()方法清空缓冲区，clear()方法会清空整个缓冲区。 
	compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面

buffer的几个属性：

* capacity 

>> 作为一个内存块，Buffer有一个固定的大小值，也叫“capacity”.你只能往里写capacity个byte、long，char等类型。一旦Buffer满了，
需要将其清空才能继续写数据进buffer。 

* position (0~capacidy-1)

>> 当你写数据到Buffer中时，position表示当前的位置。初始的position值为0.当一个byte、long等数据写到Buffer后， position会向前移动到
下一个可插入数据的Buffer单元。position最大可为capacity – 1。 当读取数据时，也是从某个特定位置读。***当将Buffer从写模式切换到读模式，
position会被重置为0。***当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。 

* limit 

>> 在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。*** 写模式下，limit等于Buffer的capacity。*** 
当切换Buffer到读模式时， limit表示你最多能读到多少数据。因此，***当切换Buffer到读模式时，limit会被设置成写模式下的position值。***

buffer API

* Buffer的分配 

要想获得一个Buffer对象首先要进行分配。 每一个Buffer类都有一个allocate方法。下面是一个分配48字节capacity的ByteBuffer的例子。 
{% highlight java %}
  ByteBuffer buf = ByteBuffer.allocate(48);
{% endhighlight %}

* 向Buffer中写数据 ,有两种方式,从Channel写到Buffer和通过Buffer的put()方法写到Buffer里。
{% highlight java %}
int bytesRead = inChannel.read(buf); //read into buffer,记住如果读需先调用flip()
buf.put(127); //通过put方法写Buffer的例子
{% endhighlight %}

* flip()方法,将Buffer从写模式切换到读模式 

>>调用flip()方法会将position设回0，并将limit设置成之前position的值。 
换句话说，position现在用于标记读的位置，limit表示之前写进了多少个byte、char等 ,现在能读取多少个byte、char等。 

* 从Buffer中读取数据,有两种方式,从Buffer读取数据到Channel(写数据到channel)和使用get()方法从Buffer中读取数据。
{% highlight java %}
int bytesWritten = inChannel.write(buf); //read from buffer into channel.
byte aByte = buf.get(); //从Buffer中读取数据
{% endhighlight %}

* clear()与compact()方法,读完Buffer中的数据，需要让Buffer准备好再次被写入。可以通过clear()或compact()方法来完成。 

>>如果调用的是clear()方法，Buffer 被清空了。Buffer中的数据并未清除，只是position重置0，limit被设置成 capacity的值,
这些标记告诉我们可以从哪里开始往Buffer里写数据，新数据会覆盖旧数据。
如果Buffer中仍有未读完的有效数据，但是此时想要先向buffer写些数据，那么可以使用compact()方法。
compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，
设置成capacity。现在Buffer准备好写数据了，但是不会覆盖未读的数据，相当于断点续传。

* equals() 

>> 当满足下列条件时，表示两个Buffer相等，buffer中存储的内容一致。

* compareTo()方法

>>compareTo()方法比较两个Buffer内的元素(byte、char等)， 如果满足下列条件，则认为一个Buffer“小于”另一个Buffer： 
第一个不相等的元素小于另一个Buffer中对应的元素。前缀所有元素都相等，第一个Buffer的元素个数比另一个少。

* mark()与reset()方法 

通过调用Buffer.mark()方法，可以标记Buffer中的一个特定position。之后可以通过调用Buffer.reset()方法恢复到这个position。
{% highlight java %}
buffer.mark();    
buffer.reset();  
{% endhighlight %}

####3.	 Selector 选择器

Selector允许单线程处理多个 Channel。如果你的应用打开了多个连接（通道），但每个连接的流量都很低，使用Selector就会很高效，管理多个连接。
要使用Selector，需要向Selector注册Channel，然后调用它的select()方法，这个方法会一直阻塞到某个注册的通道有事件就绪。
一旦这个方法返回，就表明有新的事件触发，线程就可以处理这些事件，使用示例如下：


* Selector的创建

{% highlight java %}
Selector selector = Selector.open();  
{% endhighlight %}

* 向Selector注册通道 

为了将Channel和Selector配合使用，必须将channel注册到selector上。通过SelectableChannel.register()方法来实现.
***与Selector一起使用时，Channel必须处于非阻塞模式下。这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式。而socket通道都可以。***
{% highlight java %}
channel.configureBlocking(false);  
SelectionKey key = channel.register(selector, Selectionkey.OP_READ);  
{% endhighlight %} 
***注意register()方法的第二个参数,这是一个“interest集合”，意思是在通过Selector监听Channel时对什么事件或一组事件感兴趣。***

* 可以监听如下四种不同类型的事件：

监听事件|事件常量|是否就绪
Connect|SelectionKey.OP_CONNECT|key.isAcceptable()
Accept|SelectionKey.OP_ACCEPT|key.isConnectable();
Read|SelectionKey.OP_READ|key.isReadable();  
Write|SelectionKey.OP_WRITE |key.isWritable(); 

* 选择通道
 
>>select()	阻塞到至少有一个通道在你注册的事件上就绪了。 

>>select(long timeout)	和select()一样，除了最长会阻塞timeout毫秒(参数)。 

>>selectNow()不会阻塞，不管什么通道就绪都立刻返回

* selectedKeys() 获取就绪事件，从而获取通道信息
{% highlight java %}
Set selectedKeys = selector.selectedKeys();  
Iterator keyIterator = selectedKeys.iterator();  
while(keyIterator.hasNext()) {  
    SelectionKey key = keyIterator.next();  
    if(key.isAcceptable()) {  
         // a connection was accepted by a ServerSocketChannel.  
         SocketChannel sc=(SocketChannel)key.channel();  
         sc.configureBlocking(false);  
         //对监听到连接事件的通道再注册读事件
		 sc.register(selector, SelectionKey.OP_READ);
         sc.finishConnect();   
    } else if (key.isConnectable()) {  
        // a connection was established with a remote server.  
    } else if (key.isReadable()) {  
         // a channel is ready for reading  
    } else if (key.isWritable()) {  
        // a channel is ready for writing  
    }
   }
{% endhighlight %} 


	  
	 
	
	 





  



 