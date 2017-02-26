---
layout: post
title:     记一次并发写hashmap产生的死循环
date:    2017-02-25 14:22:11
tags:
- java
---

<h3>问题</h3>

<p>发版本后以前每天正常工作的自检程序，突然不灵了。</p>

<h3>分析过程</h3>

<p>通过top查看到进程中的线程，有10条线程处于100%cpu消耗状态，而且没有丝毫降低迹象。</p>

<pre><code class="java">
Tasks: 976 total,  11 running, 965 sleeping,   0 stopped,   0 zombie 
Cpu(s): 45.1%us,  0.4%sy,  0.0%ni, 54.5%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  32957896k total, 31594400k used,  1363496k free,    36024k buffers
Swap: 33554428k total,  1154544k used, 32399884k free, 15165192k cached

PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
100674 apps      20   0 22.2g 9.3g  10m R 100.2 29.6 749:09.60 java
100675 apps      20   0 22.2g 9.3g  10m R 100.2 29.6 748:47.30 java
101117 apps      20   0 22.2g 9.3g  10m R 100.2 29.6 748:43.89 java
101121 apps      20   0 22.2g 9.3g  10m R 100.2 29.6 748:51.20 java
101122 apps      20   0 22.2g 9.3g  10m R 100.2 29.6 748:40.47 java
101128 apps      20   0 22.2g 9.3g  10m R 99.9 29.6 748:22.78 java
101130 apps      20   0 22.2g 9.3g  10m R 99.9 29.6 749:15.14 java
104951 apps      20   0 22.2g 9.3g  10m R 99.9 29.6 749:44.18 java
112945 apps      20   0 22.2g 9.3g  10m R 99.9 29.6 749:25.52 java
112948 apps      20   0 22.2g 9.3g  10m R 99.9 29.6 749:22.82 java
 70509 apps      20   0 22.2g 9.3g  10m R 48.3 29.6   2:19.03 java
 98167 apps      20   0 22.2g 9.3g  10m S  6.9 29.6  82:42.20 java
 98467 apps      20   0 22.2g 9.3g  10m S  1.0 29.6   4:16.54 java
 98468 apps      20   0 22.2g 9.3g  10m S  1.0 29.6   4:17.88 java
 98469 apps      20   0 22.2g 9.3g  10m S  1.0 29.6   4:18.04 java
 98125 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:28.49 java
 98141 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   7:09.18 java
 98495 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:25.76 java
 98596 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:03.13 java
 98723 apps      20   0 22.2g 9.3g  10m S  0.3 29.6  14:48.53 java 
 98805 apps      20   0 22.2g 9.3g  10m S  0.3 29.6  14:50.73 java
 98864 apps      20   0 22.2g 9.3g  10m S  0.3 29.6  14:52.03 java
 98889 apps      20   0 22.2g 9.3g  10m S  0.3 29.6  14:49.12 java
 99071 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:11.06 java
 99077 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:11.15 java
 99081 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:11.07 java
 99085 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:11.08 java
 99086 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:43.14 java
 99087 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:43.12 java
 99089 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:43.15 java
 99091 apps      20   0 22.2g 9.3g  10m S  0.3 29.6   0:43.16 java
</code>
</pre>

<p>接下来通过jstack获取到的线程信息发现，发现这10条线程全是参与自检的10个线程，他们都处于同一行代码，而且线程状态是running，很有可能死循环了。</p>

<pre>
    <code>
at java.util.HashMap.put(HashMap.java:391)
at java.util.HashSet.add(HashSet.java:217)
</code>
</pre>

<p>打开jdk中src.zip里java.util.HashMap.java的391行代码一看，果然是个循环，为什么会死循环呢？</p>

<pre><code class="java">
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for (Entry&lt;K,V&gt; e = table[i]; e != null; e = e.next) { //这就是391行
        Object k;
        if (e.hash == hash &amp;&amp; ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
</code>
</pre>