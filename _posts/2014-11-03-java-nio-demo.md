---
layout: post
title: JAVA NIO 使用示例
category: [并发编程,NIO]
---
[参考：《分布式JAVA应用》 ](http://book.douban.com/subject/4848587/)

####1.	 TCP/IP+NIO 客户端和服务器端代码片段
Channel有SocketChannel和ServerSocketChannel两种，SocketChannel用于建立连接、监听事件及操作读写，
ServerSocketChannel用于监听端口及监听连接事件；程序通过Selector来获取是否有要处理的事件。

* #####  TCP/IP+NIO 客户端代码示例

注意：注册感兴趣的IO读事件时，通常不直接注册写事件，在发送缓冲区未满的情况下，一直是可写的，因此如注册了写事件，
而又不用写数据，很容易造成CPU消耗100%的现象  
{% highlight java %}
SocketChannel channel=SocketChannel.open();  
channel.configureBlocking(false);  // 设置为非阻塞模式  
channel.connect(SocketAddress);    
Selector selector=Selector.open();  
channel.register(selector,SelectionKey.OP_CONNECT); //注册监听连接事件
int nKeys=selector.select() //这个无参方法是阻塞的，直到有事件到达
SelectionKey sKey=null;  
if(nKeys>0){  
    Set<SelectionKey> keys=selector.selectedKeys();  
	for(SelectionKey key:keys){  
	    if(key.isConnectable()){  
	        SocketChannel sc=(SocketChannel)key.channel();  
	        sc.configureBlocking(false);  
	        //对监听到连接事件的通道再注册读事件
			sKey = sc.register(selector, SelectionKey.OP_READ);
	        sc.finishConnect();   
		}else if(key.isReadable()){  
			//通道中有了服务器新数据可以读了，把通道中的数据通过buffer都读出来。
			//不要理解错这里的1024只是初始长度，容量不够会自动拉长。 
	        ByteBuffer buffer=ByteBuffer.allocate(1024); 
	        SocketChannel sc=(SocketChannel) key.channel();  
	        int readBytes=0;  
            int ret=0;  
			while((ret=sc.read(buffer))>0){  
                readBytes+=ret;  
            }  
            System.out.print(readBytes); 
            buffer.flip();//翻转buffer把数据读出来
			while(buffer.hasRemaining()){  
				System.out.print((char) buf.get());  
			}
            buffer.clear();//重置buffer位置，以备继续装数据
	    }else if(key.isWritable()){  
	    	// 取消对OP_WRITE事件的注册
	        key.interestOps(key.interestOps()& (~SelectionKey.OP_WRITE));   
	        SocketChannel sc=(SocketChannel) key.channel();  
	        //把buffer中数据写进通道
			int writtenedSize=sc.write(ByteBuffer); 
			// 如未写入，则继续注册感兴趣的OP_WRITE事件   
			if(writtenedSize==0){    
			    key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);  
			}  
		}  
      }  
    keys.clear();  
 }  
 {% endhighlight %} 
如上所述，对于要写入的流，可直接调用channel.write来完成，只有在写入未成功时才要注册OP_WRITE事件
{% highlight java %}
int wSize=channel.write(ByteBuffer);  
if(wSize==0){  
    key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);  
} 
{% endhighlight %} 

* #####  TCP/IP+NIO 服务端代码示例
NIO是典型的Reactor模式的实现，通过注册感兴趣的事件及扫描是否有感兴趣的事件发生，从而做相应的动作。
{% highlight java %}
ServerSocketChannel ssc=ServerSocketChannel.open();  
ServerSocket serverSocket=ssc.socket();    
serverSocket.bind(new InetSocketAddress(port));  
ssc.configureBlocking(false);  
Selector selector=Selector.open();   
ssc.register(selector, SelectionKey.OP_ACCEPT); 

int nKeys=selector.select() //这个无参方法是阻塞的，直到有事件到达
SelectionKey sKey=null;  
if(nKeys>0){  
    Set<SelectionKey> keys=selector.selectedKeys();  
	for(SelectionKey key:keys){  
		if(key.isAcceptable()){  
		    ServerSocketChannel server=(ServerSocketChannel)key.channel();  
		    SocketChannel sc=server.accept();  
		    if(sc==null)continue;  
			sc.configureBlocking(false);  
			sc.register(selector,SelectionKey.OP_READ);  
		}else if(key.isReadable()){
			//TODO 与客户端代码类似
		}else if(key.isWritable()){
			//TODO 与客户端代码类似
		}
	}
 }
{% endhighlight %} 

  



 