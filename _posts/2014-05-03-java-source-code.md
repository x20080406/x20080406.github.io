---
layout: post
title:  ����JDK����javaԴ��
date:   2014-05-03 00:43:57
tags:
- java
---

##ΪʲôҪ����jdk�µ�src.zip##
����鿴ĳ���Դ��ʱ������ʹ����jul(java.util.logging)������־�������õ���logback�����Ǻ�����jul-to-slf4j��ע����SLF4JBridgeHandler����Ȼslf4j�Ѿ���������־������Ȼ����ԭ����jul��Ȼ�ڴ�ӡ��־������ע��������handler�Ƶ�(�������ֵ�ȷע����2��handler������Ϊ$JAVA_HOME/jre/lib/logging.properties�����֡����ļ��е�.level=INFO����ΪOFF����ͣ��)��Ϊ�˲�ԭ���һ��һ������ȥ��jul�ڸ�ʲô�����Ƿ��־ֲ������ǿ������ġ���ȥ�������¹���ʦ��֪��ԭ��--��Ҫ���±���Դ�롣Ŀǰ��û�з����������������

````SLF4JBridgeHandler.install();//ע��jul handler````

�����׾���rt.jar����ʱû�м���**-g**�������ɵ�����Ϣ��

��ϸ�����[����](http://hllvm.group.iteye.com/group/topic/25798)

##���벽��##
###1.��ѹԴ�벢��Ҫ������ļ�д�뵽һ��file��### 
```
cd /opt/jdk
mkdir -p tmp/src
mkdir tmp/classes
mkdir tmp/rt
unzip src.zip -d tmp/src
cd tmp/src
find -name *.java|awk '{print substr($0,3)}' > filename.txt 
```

###2.����Դ��###

����ô�඼��Ϊ�˼�**\-g**����������ʱ����**-XDignore.symbol.file=true**��**-Xlint:none**���Ծ���

```
javac -XDignore.symbol.file=true -g -Xlint:none -d ../classes -cp ../jre/lib/rt.jar:../jre/lib/tools.jar @filename.txt
```

###3.���´��rt.jar###
```
cd ../rt
cp $JAVA_HOME/jre/lib/rt.jar .
jar -xf rt.jar 
cp -rf ../classes/* .
rm rt.jar
jar cvf rt.jar *
mv $JAVA_HOME/jre/lib/rt.jar $JAVA_HOME/jre/lib/rt.jar.bak
mv rt.jar $JAVA_HOME/jre/lib/
```

ok�����´�eclipse������Դ�����ܿ���������Ϣ�ˡ�����ͼ
![variables!!!](http://i63.photobucket.com/albums/h134/x20080406/eclipse-debug_zps345dd186.png)
##�������뷽��

ʹ��ant���룺[����](http://hllvm.group.iteye.com/group/topic/25839)