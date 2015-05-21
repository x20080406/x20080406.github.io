---
layout: post
title:  ����Gradle��DynamicTask����Ӧ�������ò���
date:   2015-05-21 23:40:30
tags:
- gradle
---

����ͨ���ǽ����ò���д�뵽�ܶ������ļ��Ȼ��ʹ��spring�ṩ��ռλ���ȹ����������ء�Gradle��֧��(��̬task)[https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#N1029F]����������gradle�ڱ���ʱֱ�ӽ������ļ���������滻�������ļ����Ҳ��gradle��ǿ��֮������build.gradle��дgroovy�ű���
�ҵ������ļ���

####build.gradle�еĲ��ִ���
```groovy
def envs=['proc','test'] //���廷��

//����task
envs.each{ env ->
	task "build_${env}" (dependsOn:build) << {
		copy{
			from "$buildDir/resources/main"
			into ("$buildDir/beans")
			def myProps = new Properties()
			file("${env}.properties").withInputStream{
				myProps.load(it);   
			}
			filter(org.apache.tools.ant.filters.ReplaceTokens, tokens: myProps)
		}
	}
}
```

####proc.properties
```
key1=procval
```
####test.properties
```
key1=testval
```
####spring�����ļ�
```xml
<bean id="mybean1" class="com.test.MyClass1" >
    <property name="myprop1" value="@key1@" />
</bean>
```

�������Ŀ��ִ��**gradle build_proc**����**gradle build_test**