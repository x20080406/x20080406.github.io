---
layout: post
title:  ��Դ�����ΪʲôNetty4.0.x��ChannelOutboundHandler��Ҫ����ChannelInboundHandler֮ǰ
date:   2014-08-25 12:08:47
tags:
- java 
- netty
---

##��Դ�����ΪʲôNetty4.0.x��ChannelOutboundHandler��Ҫ����ChannelInboundHandler֮ǰ��

####����
���ѧϰnetty�������channelִ��˳���ʱ��д��һ�����ӡ��ɷ���ChannelOutboundHandler�����޷����á����˺ܶ����϶�˵����ִ��˳����Servlet������������servletȥ��⣬�Ǿ��ϵ��ˣ�

####����
�ڴ�����д����������»ᰴ��[�ٷ��ĵ�](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)��˵���Ĳ���ִ�С���������������˵��������inboundhandlerִ������֮����ȥִ��outboundhandler������Ȼ����д����Ҳ�ð������ַ�ʽȥд����Ȼ����Ͳ����ˡ�֮����д���ĵ�����Ϊ�ҵĳ�����˴���ԭ�򡣺Ǻǡ���
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

####��������
netty�ڲ�������channel����װ��Ϊһ��ChannelHandlerContext(Ĭ����DefaultChannelHandlerContext)��Ȼ����һ��__˫������__��š�ͷ��β�ֱ������������Handler���ֱ����HeadHandler��TailHandler(��DefaultChannelPipeline������п����ҵ�)��������ͨ��channelд������ʱ��ͼ��ߵ�˳��ʼִ�У�ֱ�����һ��InboundHandler��ִ��Ϊֹ�����統ǰinboundhandler�������һ��Ԫ�أ���ô����Ͳ����ˡ���ͨ��ChannelHandlerContext��write����ת������Outboundhandlerʱ����ǰinboundhandler�ĺ����outboundhandler��Զ���ᱻִ�У�Ϊʲô�أ��ÿ�������findContextInbound��findContextOutbound����������������������ʱ����__˳�ű�������(findContextInbound)__��ֱ�����һ��InboundHandler���ҵ�ʱ�����κ�һ��Inboundhandler����ChannelHandlerContext.write����ʱ������ǰ�������һ��Inboundhandler�Ļ���OutboundHandlerִ����InboundHandler�������ִ�У�����ǰChannelHandlerContext�ͻ�ȥִ�е�ǰԪ��ǰ���OutboundHandler������ִ��˳�����__�ӵ�ǰԪ�ط��Ų��ҵ�ǰ�����Ӻ���ǰ��findContextOutbound��__���������һ��Ͳ�����������ͼ���Ǹ�˳���ˡ�

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

####���Դ���
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
		//out2���ᱻִ��
		cp.addLast(in1);
		cp.addLast(out1);
		cp.addLast(in2);
		cp.addLast(out2);
		*/
		
		/*
		//out1,out2�����ᱻִ��
		cp.addLast(in1);
		cp.addLast(in2);
		cp.addLast(out1);
		cp.addLast(out2);
		 */
		
		/*
		//out1���ᱻִ��
		cp.addLast(in1);
		cp.addLast(out2);
		cp.addLast(in2);
		cp.addLast(out1);
		*/

		
		//out1��out2���ᱻִ��
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
		log.info("������һ��ִ��");
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
        //result.resetReaderIndex();//���ǰ��������ݣ���Ҫ���ö�����
        byte[] bytes = new byte[data.readableBytes()];  
        data.readBytes(bytes);  
        System.out.println("�ͻ��˴�����������:" + new String(bytes));  
        
        
  
        ctx.write("Object");  //notify outboundhandler!
        //outbound�в���ȡ���ݣ���ô�˴���Ҫ�ͷ���Դ����Ȼ���ڴ�й©Ŷ
        data.release();//ReferenceCountUtil.release(msg);
        
        //ctx.fireChannelRead(msg);����û��inboundhandler����Ҫfire��
	}
	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		super.channelReadComplete(ctx);
		log.info("�����ڶ�");
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
        String response = "Server said��"+ctx.channel().remoteAddress().toString()+"\n";  
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

####�ͻ�����telnet�ͺ���
telnet 127.0.0.1 8080��Ȼ��������룡