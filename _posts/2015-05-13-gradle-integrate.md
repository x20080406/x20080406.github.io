---
layout: post
title:  将maven项目整合到Gradle中
date:   2015-05-13 15:33:25
tags:
- java
- gradle 
---

<h2 id="mavengradle">将maven项目转换到Gradle</h2>
<p>目前公司项目开始用Gradle构建。最近开始将RPC框架作为子项目合并到公司的项目里去。RPC框架一开始使用maven构建所以合并起来非常简单.除了将依赖格式转换为Gradle依赖写发以外基本上无需其他配置就能合并进去。</p>
<h2 id="">创建子项目</h2>
<p>RPC框架是一个java项目。在Gradle项目中创建好子项目目录RPC（空文件夹一个），并往里面添加一个名为<strong>build.gradle</strong>空文件。将maven项目中的src目录拷贝到RPC目录下。然后在Gradle的根项目的settings文件夹中添加一行。语句将RPC项目添加到Gradle项目中。</p>
<pre><code>include &quot;:RPC&quot;
</code></pre>

<p>就这样一个子项目就创建好了。但由于RPC框架分很多模块，所以一个模块就演变为了一个子项目。RPC目录实际上不是一个子项目，里面存放的目录才是子项目。我在gradle.settings中添加的如下：</p>
<pre><code>include &quot;RPC:RPC-client&quot;,&quot;RPC:RPC-common&quot;,&quot;RPC:RPC-server&quot; ,&quot;RPC:RPC-facade&quot;,&quot;RPC:RPC-impl&quot;
</code></pre>

<p>所以RPC目录中有5个子项目,如上所示。另外我将公共依赖以及一些公共task提取出来放置到一个公共文件中，然后在子项目中导入这个文件。我将一些公共依赖，如slfj-api，junit，以及拷贝项目依赖的task放置到名为RPC.build.gradle的文件中去。
这只是一个公共配置，任何子模块通过命令 <strong>apply from: "$rootProject.projectDirPath/RPC/RPC.build.gradle"</strong>将文件导入即可使用这些配置及task。项目中所采用的开源组件的版本以及一些其他配置是在根项目中定义的，不需要引入，直接通过内置变量引用即可。</p>
<pre><code>apply plugin: &quot;java&quot;
version = &quot;0.1&quot;
repositories {
    //mavenLocal()//本地maven库中安装了一些项目，Gradle不将它cache起来而是直接引用，导致调试时无法跟中源码。注掉就OK了。
    maven {
        url &quot;$rootProject.buildConf.build.mavenRepoUrl&quot;
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
    compile (&quot;io.netty:netty-all:${rootProject.buildConf.depVersion.RPC.netty}&quot;){
        exclude module: 'slf4j-api'//不要依赖slf4j-api，下面已经有依赖
    }
    //将jul，jcl，log4j全部交给logback处理
    compile &quot;org.slf4j:jul-to-slf4j:${rootProject.buildConf.depVersion.slf4j}&quot;
    compile &quot;org.slf4j:jcl-over-slf4j:${rootProject.buildConf.depVersion.slf4j}&quot;
    compile &quot;org.slf4j:log4j-over-slf4j:${rootProject.buildConf.depVersion.slf4j}&quot;
    compile &quot;org.slf4j:slf4j-api:${rootProject.buildConf.depVersion.slf4j}&quot;
    compile &quot;ch.qos.logback:logback-classic:${rootProject.buildConf.depVersion.logback}&quot;
    //guava
    compile &quot;com.google.guava:guava:${rootProject.buildConf.depVersion.RPC.guava}&quot;
    compile 'com.google.code.findbugs:jsr305:3.0.0'
    //junit
    testCompile &quot;junit:junit:${rootProject.buildConf.depVersion.test.junit}&quot;
}

/**
 * 拷贝compile级别的以来到buildDir下的output/lib目录下
 */
task copyLibs(type: Copy) {
    into &quot;$buildDir/output/libs&quot;
    from configurations.compile
}
</code></pre>

<h2 id="_1">新建子项目</h2>
<p>以RPC-client子项目为例。在RPC文件夹下建立目录RPC-client，并为他添加一个build.gradle文件，文件大致内容如下：</p>
<pre><code>apply from: &quot;$rootProject.projectDirPath/RPC/RPC.build.dependencies.gradle&quot;

dependencies {
    compile project(':CORE:CORE-COMMON')
    compile project(':RPC:RPC-common')
    compile project(':RPC:RPC-facade')
}
/**
 * 编译客户端。由于有些项目为使用任何构建工具，需要将项目中所需要的依赖全部copy出来
 */
task buildClient (dependsOn:[clean,jar,copyLibs]) &lt;&lt; {
    copy {
        from &quot;build/libs&quot;
        into &quot;$buildDir/output/libs&quot;
    }
}
jar.shouldRunAfter clean
copyLibs.shouldRunAfter jar

task buildClientZip(type: Zip,dependsOn:buildClient) {
    from &quot;$buildDir/dist/rpc-client.zip&quot;
}

</code></pre>

<p>就这样，一个新的子项目就添加到一个大项目中了</p>
<h3 id="task">常用task</h3>
<p>生成项目中的源文件目录。sourceSets是在java插件中定义的，需先引入插件<strong>apply plugin: 'java'</strong>和<strong>apply plugin: 'scala'</strong></p>
<pre><code>apply plugin: &quot;java&quot;
apply plugin: &quot;scala&quot;

/**
 * 已经约定好了。通常不需要指定
 */
sourceSets{
    main{
        java{
            srcDirs=[&quot;$projectDir/src/main/java&quot;]
        }
        scala{
            srcDirs=[&quot;$projectDir/src/main/scala&quot;]  
        }
    }
}
task printSourceSet &lt;&lt; {
    sourceSets.all { set -&gt;
        set.allSource.srcDirs.each { dir -&gt;
            //dir.mkdirs()
            println dir 
        }
    }
}

</code></pre>

<h3 id="java">打包java项目</h3>
<p>引入application插件<strong>apply plugin: "application"</strong></p>
<pre><code>apply from: &quot;$rootProject.projectDirPath/RPC/RPC.build.gradle&quot;
apply plugin: &quot;application&quot;

//设置应用名
applicationName = &quot;fzsRpcServer&quot;
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
    include &quot;**/**/*.class&quot;
}

/**
 * 处理启动脚本&lt;br/&gt;
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
task mybatisGenerate &lt;&lt; {
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
task cleanDistribution(dependsOn: clean) &lt;&lt; {
    def distDir = file(buildDir.getParent() + &quot;/dist&quot;)
    if (distDir.exists()) {
        delete distDir.listFiles()
    }
}

/**
 * 编译可执行的服务器
 */
task buildServer(dependsOn: ['cleanDistribution', 'build', 'jar', 'startScripts', 'copyLibs']) &lt;&lt; {
    copy {
        from &quot;build/libs&quot;
        from &quot;$buildDir/output/libs&quot;
        into &quot;build/dist/$applicationName/lib&quot;
    }
    copy {
        from &quot;$buildDir/scripts&quot;
        into &quot;build/dist/$applicationName/bin&quot;
    }
    copy {
        from &quot;build/resources/main&quot;
        into &quot;build/dist/$applicationName/config&quot;
    }
    print &quot;编译结果：$buildDir/dist/$applicationName&quot;
}

//处理执行顺序
buildServer.shouldRunAfter cleanDistribution

/**
 * 生成压缩包
 */
task buildServerZip(type: Zip,dependsOn:buildServer) {
    print  &quot;$buildDir/dist/$applicationName&quot;
    from &quot;$buildDir/dist/$applicationName&quot;
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
    mybatisGenDependencies &quot;com.oracle:ojdbc:${rootProject.buildConf.depVersion.oracleOjdbc}&quot; //jdbc driver
}
</code></pre>