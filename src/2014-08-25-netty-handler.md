---
layout: post
title:  从源码分析为什么Netty4.0.x的ChannelOutboundHandler需要放在ChannelInboundHandler之前
date:   2014-08-25 12:08:47
tags:
- java 
- netty
---

####起因
最近学习netty，在理解channel执行顺序的时候写了一个例子。可发现ChannelOutboundHandler死活无法调用。看了很多资料都说它的执行顺序像Servlet，如果真把它按servlet去理解，那就上当了！

####资料
在代码书写正常的情况下会按照[官方文档](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)所说明的步骤执行。大致如网上资料说的那样。inboundhandler执行完了之后再去执行outboundhandler。（当然我们写代码也得按照这种方式去写。不然程序就不对了。之所以写此文档是因为我的程序出了错找原因。呵呵～）
```
                                                 I/O Request
                                            via Channel or
                                        ChannelHandlerContext
                                                      |
  +---------------------------------------------------+---------------+
  |                           ChannelPipeline         |               |
  |                                                  \|/              |
  |    +---------------------+            +-----------+----------+    |
  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  .               |
  |               .                                   .               |
  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
  |        [ method call]                       [method call]         |
  |               .                                   .               |
  |               .                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  |               |                                  \|/              |
  |    +----------+----------+            +-----------+----------+    |
  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
  |    +----------+----------+            +-----------+----------+    |
  |              /|\                                  |               |
  +---------------+-----------------------------------+---------------+
                  |                                  \|/
  +---------------+-----------------------------------+---------------+
  |               |                                   |               |
  |       [ Socket.read() ]                    [ Socket.write() ]     |
  |                                                                   |
  |  Netty Internal I/O Threads (Transport Implementation)            |
  +-------------------------------------------------------------------+

```
  
####Forwarding an event to the next handler
````

	Inbound event propagation methods:
        ChannelHandlerContext.fireChannelRegistered()
        ChannelHandlerContext.fireChannelActive()
        ChannelHandlerContext.fireChannelRead(Object)
        ChannelHandlerContext.fireChannelReadComplete()
        ChannelHandlerContext.fireExceptionCaught(Throwable)
        ChannelHandlerContext.fireUserEventTriggered(Object)
        ChannelHandlerContext.fireChannelWritabilityChanged()
        ChannelHandlerContext.fireChannelInactive()
        ChannelHandlerContext.fireChannelUnregistered()
    Outbound event propagation methods:
        ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
        ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
        ChannelHandlerContext.write(Object, ChannelPromise)
        ChannelHandlerContext.flush()
        ChannelHandlerContext.read()
        ChannelHandlerContext.disconnect(ChannelPromise)
        ChannelHandlerContext.close(ChannelPromise)
        ChannelHandlerContext.deregister(ChannelPromise)

````

####分析问题
netty内部将所有channel都包装成为一个ChannelHandlerContext(默认是DefaultChannelHandlerContext)，然后用一个__双向链表__存放。头和尾分别是两个特殊的Handler。分别叫做HeadHandler和TailHandler(在DefaultChannelPipeline这个类中可以找到)。当请求通过channel写入数据时上图左边的顺序开始执行，直到最后一个InboundHandler被执行为止。假如当前inboundhandler不是最后一个元素，那么问题就产生了。当通过ChannelHandlerContext的write方法转发请求到Outboundhandler时，当前inboundhandler的后面的outboundhandler永远不会被执行，为什么呢？得看看他的findContextInbound和findContextOutbound这两个方法！当有请求来时，会__顺着遍历链表(findContextInbound)__，直到最后一个InboundHandler被找到时，当任何一个Inboundhandler调用ChannelHandlerContext.write方法时（若当前不是最后一个Inboundhandler的话，OutboundHandler执行完InboundHandler还会继续执行），当前ChannelHandlerContext就会去执行当前元素前面的OutboundHandler。它的执行顺序就是__从当前元素反着查找当前链表（从后向前，findContextOutbound）__。理解了这一块就不难明白上面图中那个顺序了。

````

	//io.netty.channel.DefaultChannelHandlerContext.findContextInbound()

    private DefaultChannelHandlerContext findContextInbound() {
        DefaultChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!(ctx.handler instanceof ChannelInboundHandler));
        return ctx;
    }

	//io.netty.channel.DefaultChannelHandlerContext.findContextOutbound()

    private DefaultChannelHandlerContext findContextOutbound() {
        DefaultChannelHandlerContext ctx = this;
        do {
            ctx = ctx.prev;
        } while (!(ctx.handler instanceof ChannelOutboundHandler));
        return ctx;
    }

````

####测试代码
NcfChannelInitializer.java
```

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOutboundHandlerAdapter;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.ChannelPromise;
import io.netty.channel.socket.SocketChannel;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class NcfChannelInitializer extends ChannelInitializer<SocketChannel> {
	private Logger log = LoggerFactory.getLogger(NcfChannelInitializer.class);
	private static InboundHandler1 in1 = new InboundHandler1();
	private static InboundHandler2 in2 = new InboundHandler2();
	private static OutboundHandler1 out1 = new OutboundHandler1();
	private static OutboundHandler2 out2 = new OutboundHandler2();
	@Override
	protected void initChannel(SocketChannel ch) throws Exception {
		ChannelPipeline cp = ch.pipeline();
		
		log.info("init channels");
		/*
		//out2不会被执行
		cp.addLast(in1);
		cp.addLast(out1);
		cp.addLast(in2);
		cp.addLast(out2);
		*/
		
		/*
		//out1,out2都不会被执行
		cp.addLast(in1);
		cp.addLast(in2);
		cp.addLast(out1);
		cp.addLast(out2);
		 */
		
		/*
		//out1不会被执行
		cp.addLast(in1);
		cp.addLast(out2);
		cp.addLast(in2);
		cp.addLast(out1);
		*/

		
		//out1，out2都会被执行
		cp.addLast(out1);
		cp.addLast(in1);
		cp.addLast(out2);
		cp.addLast(in2);
		
	}
}

@Sharable
class InboundHandler1 extends ChannelInboundHandlerAdapter {
	private Logger log = LoggerFactory.getLogger(InboundHandler1.class);

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg)
			throws Exception {
		log.info("InboundHandler1.channelRead");

//        ctx.write("qqq");///!!!! notify outboundhandler
        ctx.fireChannelRead(msg);  //notify next inboundhandler
	}
	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		super.channelReadComplete(ctx);
		log.info("倒数第一被执行");
	}
}

@Sharable
class InboundHandler2 extends ChannelInboundHandlerAdapter {
	private Logger log = LoggerFactory.getLogger(InboundHandler2.class);
	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg)
			throws Exception {
		log.info("InboundHandler2.channelRead");
        ByteBuf data = (ByteBuf) msg;  
        //result.resetReaderIndex();//如果前面读过数据，需要重置读索引
        byte[] bytes = new byte[data.readableBytes()];  
        data.readBytes(bytes);  
        System.out.println("客户端传过来的数据:" + new String(bytes));  
        
        
  
        ctx.write("Object");  //notify outboundhandler!
        //outbound中不读取数据，那么此处需要释放资源，不然会内存泄漏哦
        data.release();//ReferenceCountUtil.release(msg);
        
        //ctx.fireChannelRead(msg);后面没有inboundhandler，不要fire！
	}
	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		super.channelReadComplete(ctx);
		log.info("倒数第二");
	}

}

// ////////////////////////////////////

@Sharable
class OutboundHandler1 extends ChannelOutboundHandlerAdapter {
	private static Logger log = LoggerFactory.getLogger(OutboundHandler1.class);

	@Override
	public void write(ChannelHandlerContext ctx, Object msg,
			ChannelPromise promise) throws Exception {
		log.info("OutboundHandler1.write");
        String response = "Server said："+ctx.channel().remoteAddress().toString()+"\n";  
        ByteBuf buf = Unpooled.wrappedBuffer(response.getBytes());
        ctx.writeAndFlush(buf);  
        buf.release();
	}
	
}

@Sharable
class OutboundHandler2 extends ChannelOutboundHandlerAdapter {
	private static Logger log = LoggerFactory.getLogger(OutboundHandler2.class);

	@Override
	public void write(ChannelHandlerContext ctx, Object msg,
			ChannelPromise promise) throws Exception {
		log.info("OutboundHandler2.write");
		super.write(ctx, msg, promise);
	}
}

```

NcfServer.java
```

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NcfServer {
	public static void main(String[] args) {
		EventLoopGroup bossGroup = new NioEventLoopGroup();
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup, workerGroup)
					.channel(NioServerSocketChannel.class)
					.childHandler(new NcfChannelInitializer())
					.option(ChannelOption.SO_BACKLOG, 128)
					.childOption(ChannelOption.SO_KEEPALIVE, true);

			ChannelFuture f = b.bind(8080).sync();

			f.channel().closeFuture().sync();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			workerGroup.shutdownGracefully();
			bossGroup.shutdownGracefully();
		}
	}
}

```

####客户端用telnet就好了
telnet 127.0.0.1 8080，然后随便输入！