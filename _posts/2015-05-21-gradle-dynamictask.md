---
layout: post
title:  利用Gradle的DynamicTask来适应各种配置参数
date:   2015-05-21 23:40:30
tags:
- gradle
---
<p>我们通常是将配置参数写入到很多属性文件里，然后使用spring提供的占位符等工具类来加载。Gradle中支持<a href="https://docs.gradle.org/current/userguide/tutorial_using_tasks.html#N1029F">动态task</a>，可以利用gradle在编译时直接将属性文件里的内容替换到配置文件里。这也是gradle的强大之处，在build.gradle中写groovy脚本。以下是我的配置文件。</p>
<h4 id="buildgradle">build.gradle中的部分代码</h4>
<pre><code class="groovy">def envs=['proc','test'] //定义环境

//定义task
envs.each{ env -&gt;
    task &quot;build_${env}&quot; (dependsOn:build) &lt;&lt; {
        copy{
            from &quot;$buildDir/resources/main&quot;
            into (&quot;$buildDir/beans&quot;)
            def myProps = new Properties()
            file(&quot;${env}.properties&quot;).withInputStream{
                myProps.load(it);   
            }
            filter(org.apache.tools.ant.filters.ReplaceTokens, tokens: myProps)
        }
    }
}
</code></pre>

<h4 id="procproperties">proc.properties</h4>
<pre><code>key1=procval
</code></pre>

<h4 id="testproperties">test.properties</h4>
<pre><code>key1=testval
</code></pre>

<h4 id="spring">spring配置文件</h4>
<pre><code class="xml">&lt;bean id=&quot;mybean1&quot; class=&quot;com.test.MyClass1&quot; &gt;
    &lt;property name=&quot;myprop1&quot; value=&quot;@key1@&quot; /&gt;
&lt;/bean&gt;
</code></pre>

<p>最后在项目中执行<strong>gradle build_proc</strong>或者<strong>gradle build_test</strong></p>