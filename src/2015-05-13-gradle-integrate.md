---
layout: post
title:  将maven项目整合到Gradle中
date:   2015-05-13 15:33:25
tags:
- java
- gradle 
---

##将maven项目转换到Gradle
目前公司项目开始用Gradle构建。最近开始将RPC框架作为子项目合并到公司的项目里去。RPC框架一开始使用maven构建所以合并起来非常简单.除了将依赖格式转换为Gradle依赖写发以外基本上无需其他配置就能合并进去。

##创建子项目
RPC框架是一个java项目。在Gradle项目中创建好子项目目录RPC（空文件夹一个），并往里面添加一个名为__build.gradle__空文件。将maven项目中的src目录拷贝到RPC目录下。然后在Gradle的根项目的settings文件夹中添加一行。语句将RPC项目添加到Gradle项目中。
```
include ":RPC"
```
就这样一个子项目就创建好了。但由于RPC框架分很多模块，所以一个模块就演变为了一个子项目。RPC目录实际上不是一个子项目，里面存放的目录才是子项目。我在gradle.settings中添加的如下：
```
include "RPC:RPC-client","RPC:RPC-common","RPC:RPC-server" ,"RPC:RPC-facade","RPC:RPC-impl"
```
所以RPC目录中有5个子项目,如上所示。另外我将公共依赖以及一些公共task提取出来放置到一个公共文件中，然后在子项目中导入这个文件。我将一些公共依赖，如slfj-api，junit，以及拷贝项目依赖的task放置到名为RPC.build.gradle的文件中去。
这只是一个公共配置，任何子模块通过命令 __apply from: "$rootProject.projectDirPath/RPC/RPC.build.gradle"__将文件导入即可使用这些配置及task。项目中所采用的开源组件的版本以及一些其他配置是在根项目中定义的，不需要引入，直接通过内置变量引用即可。
```
apply plugin: "java"
version = "0.1"
repositories {
    //mavenLocal()//本地maven库中安装了一些项目，Gradle不将它cache起来而是直接引用，导致调试时无法跟中源码。注掉就OK了。
    maven {
        url "$rootProject.buildConf.build.mavenRepoUrl"
    }
    mavenCentral()
}

configurations {
    all*.exclude group: 'commons-logging', module: 'commons-logging'//所有项目都不要依赖commons-logging
}

/**
 * 公共依赖
 */
dependencies {
    compile ("io.netty:netty-all:${rootProject.buildConf.depVersion.RPC.netty}"){
        exclude module: 'slf4j-api'//不要依赖slf4j-api，下面已经有依赖
    }
    //将jul，jcl，log4j全部交给logback处理
    compile "org.slf4j:jul-to-slf4j:${rootProject.buildConf.depVersion.slf4j}"
    compile "org.slf4j:jcl-over-slf4j:${rootProject.buildConf.depVersion.slf4j}"
    compile "org.slf4j:log4j-over-slf4j:${rootProject.buildConf.depVersion.slf4j}"
    compile "org.slf4j:slf4j-api:${rootProject.buildConf.depVersion.slf4j}"
    compile "ch.qos.logback:logback-classic:${rootProject.buildConf.depVersion.logback}"
    //guava
    compile "com.google.guava:guava:${rootProject.buildConf.depVersion.RPC.guava}"
    compile 'com.google.code.findbugs:jsr305:3.0.0'
    //junit
    testCompile "junit:junit:${rootProject.buildConf.depVersion.test.junit}"
}

/**
 * 拷贝compile级别的以来到buildDir下的output/lib目录下
 */
task copyLibs(type: Copy) {
    into "$buildDir/output/libs"
    from configurations.compile
}
```
##新建子项目
以RPC-client子项目为例。在RPC文件夹下建立目录RPC-client，并为他添加一个build.gradle文件，文件大致内容如下：
```
apply from: "$rootProject.projectDirPath/RPC/RPC.build.dependencies.gradle"

dependencies {
    compile project(':CORE:CORE-COMMON')
    compile project(':RPC:RPC-common')
    compile project(':RPC:RPC-facade')
}
/**
 * 编译客户端。由于有些项目为使用任何构建工具，需要将项目中所需要的依赖全部copy出来
 */
task buildClient (dependsOn:[clean,jar,copyLibs]) << {
    copy {
        from "build/libs"
        into "$buildDir/output/libs"
    }
}
jar.shouldRunAfter clean
copyLibs.shouldRunAfter jar

task buildClientZip(type: Zip,dependsOn:buildClient) {
    from "$buildDir/dist/rpc-client.zip"
}

```
就这样，一个新的子项目就添加到一个大项目中了

###常用task
生成项目中的源文件目录。sourceSets是在java插件中定义的，需先引入插件**apply plugin: 'java'**和**apply plugin: 'scala'**
```
apply plugin: "java"
apply plugin: "scala"

/**
 * 已经约定好了。通常不需要指定
 */
sourceSets{
	main{
		java{
			srcDirs=["$projectDir/src/main/java"]
		}
		scala{
			srcDirs=["$projectDir/src/main/scala"]	
		}
	}
}
task printSourceSet << {
    sourceSets.all { set ->
        set.allSource.srcDirs.each { dir ->
        	//dir.mkdirs()
        	println dir 
        }
    }
}

```
###打包java项目
引入application插件**apply plugin: "application"**
```
apply from: "$rootProject.projectDirPath/RPC/RPC.build.gradle"
apply plugin: "application"

//设置应用名
applicationName = "fzsRpcServer"
//设置压缩包名字
archivesBaseName = applicationName
//设置启动类
mainClassName = 'com.phone580.rpc.server.NcfServer'

configurations {
    mybatisGenDependencies
}

/**
 * 测试时加入远程服务器参数。注意，只有使用Gradle才会生效！普通junit需要在vm参数中指定
 */
test{
    jvmArgs '-Dss=false','-Dhosts=10.20.1.32:23432'
}

/**
 * 不打包资源文件
 */
jar{
    include "**/**/*.class"
}

/**
 * 处理启动脚本<br/>
 * 将配置文件加入到启动文件中的classpath里并去掉lib目录
 */
startScripts {
    classpath += files('config')
    doLast {
        def windowsScriptFile = file getWindowsScript()
        def unixScriptFile = file getUnixScript()
        windowsScriptFile.text = windowsScriptFile.text.replace('%APP_HOME%\\lib\\config', '%APP_HOME%\\config')
        unixScriptFile.text  = unixScriptFile.text.replace('$APP_HOME/lib/config', '$APP_HOME/config')
    }
}

/**
 * 生成mybatis mapper文件
 */
task mybatisGenerate << {
    ant.taskdef(
            name: 'mbGeneratorTask',
            classname: 'org.mybatis.generator.ant.GeneratorAntTask',
            classpath: configurations.mybatisGenDependencies.asPath
    )
    ant.mbGeneratorTask(
            overwrite: true,
            configfile: 'src/test/resources/generatorConfig.xml',
            verbose: true)
}
**
 * 删除编译好的目录
 */
task cleanDistribution(dependsOn: clean) << {
    def distDir = file(buildDir.getParent() + "/dist")
    if (distDir.exists()) {
        delete distDir.listFiles()
    }
}

/**
 * 编译可执行的服务器
 */
task buildServer(dependsOn: ['cleanDistribution', 'build', 'jar', 'startScripts', 'copyLibs']) << {
    copy {
        from "build/libs"
        from "$buildDir/output/libs"
        into "build/dist/$applicationName/lib"
    }
    copy {
        from "$buildDir/scripts"
        into "build/dist/$applicationName/bin"
    }
    copy {
        from "build/resources/main"
        into "build/dist/$applicationName/config"
    }
    print "编译结果：$buildDir/dist/$applicationName"
}

//处理执行顺序
buildServer.shouldRunAfter cleanDistribution

/**
 * 生成压缩包
 */
task buildServerZip(type: Zip,dependsOn:buildServer) {
    print  "$buildDir/dist/$applicationName"
    from "$buildDir/dist/$applicationName"
}

/**
 * 项目依赖
 */
dependencies {
    compile project(':CORE:CORE-COMMON')
    compile project(':RPC:rpc-facade')
    compile project(':RPC:rpc-server')
    //mybatisGenerator 仅用于生成mybatismapper的tantask
    mybatisGenDependencies 'org.mybatis.generator:mybatis-generator-core:1.3.2'
    mybatisGenDependencies "com.oracle:ojdbc:${rootProject.buildConf.depVersion.oracleOjdbc}" //jdbc driver
}
```