---
layout: post
title:  利用Gradle的DynamicTask来适应各种配置参数
date:   2015-05-21 23:40:30
tags:
- gradle
---

我们通常是将配置参数写入到很多属性文件里，然后使用spring提供的占位符等工具类来加载。Gradle中支持(动态task)[https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#N1029F]，可以利用gradle在编译时直接将属性文件里的内容替换到配置文件里。这也是gradle的强大之处，在build.gradle中写groovy脚本。
我的配置文件。

####build.gradle中的部分代码
```groovy
def envs=['proc','test'] //定义环境

//定义task
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
####spring配置文件
```xml
<bean id="mybean1" class="com.test.MyClass1" >
    <property name="myprop1" value="@key1@" />
</bean>
```

最后在项目中执行**gradle build_proc**或者**gradle build_test**