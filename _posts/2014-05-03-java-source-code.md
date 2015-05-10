---
layout: post
title:  编译JDK带的java源码
date:   2014-05-03 00:43:57
tags:
- java
---

<h2 id="jdksrczip">为什么要编译jdk下的src.zip</h2>
<p>今天查看某框架源码时发现他使用了jul(java.util.logging)处理日志，而我用的是logback。于是乎加入jul-to-slf4j并注册了SLF4JBridgeHandler，虽然slf4j已经处理了日志，但仍然发现原来的jul仍然在打印日志，就像注册了两个handler似的(后来发现的确注册了2个handler。是因为$JAVA_HOME/jre/lib/logging.properties在作怪。把文件中的.level=INFO，改为OFF就消停了)。为了查原因就一步一步跟进去看jul在干什么。但是发现局部变量是看不到的。我去！问了下谷老师才知道原因--需要重新编译源码。目前还没有发现有其他解决法。</p>
<p><code>SLF4JBridgeHandler.install();//注册jul handler</code></p>
<p>归根结底就是rt.jar编译时没有加入<strong>-g</strong>参数生成调试信息。</p>
<p>详细情况看<a href="http://hllvm.group.iteye.com/group/topic/25798">这里</a></p>
<h2 id="">编译步骤</h2>
<h3 id="1file">1.解压源码并将要编译的文件写入到一个file中###</h3>
<pre><code>cd /opt/jdk
mkdir -p tmp/src
mkdir tmp/classes
mkdir tmp/rt
unzip src.zip -d tmp/src
cd tmp/src
find -name *.java|awk '{print substr($0,3)}' &gt; filename.txt 
</code></pre>

<h3 id="2">2.编译源码</h3>
<p>做这么多都是为了加<strong>-g</strong>参数。编译时加上<strong>-XDignore.symbol.file=true</strong>和<strong>-Xlint:none</strong>忽略警告</p>
<pre><code>javac -XDignore.symbol.file=true -g -Xlint:none -d ../classes -cp ../jre/lib/rt.jar:../jre/lib/tools.jar @filename.txt
</code></pre>

<h3 id="3rtjar">3.重新打包rt.jar</h3>
<pre><code>cd ../rt
cp $JAVA_HOME/jre/lib/rt.jar .
jar -xf rt.jar 
cp -rf ../classes/* .
rm rt.jar
jar cvf rt.jar *
mv $JAVA_HOME/jre/lib/rt.jar $JAVA_HOME/jre/lib/rt.jar.bak
mv rt.jar $JAVA_HOME/jre/lib/
</code></pre>

<p>ok，重新打开eclipse，进入源码后就能看到调试信息了。如下图
<img alt="variables!!!" src="http://i63.photobucket.com/albums/h134/x20080406/eclipse-debug_zps345dd186.png" /></p>
<h2 id="_1">其他编译方法</h2>
<p>使用ant编译：<a href="http://hllvm.group.iteye.com/group/topic/25839">这里</a></p>