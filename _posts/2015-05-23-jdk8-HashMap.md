---
layout: post
title:  Java8中HashMap的put值时处理hash冲突
date:   2015-05-23 16:12:55
tags:
- java 
---
<p>jdk1.8的HashMap的Hash</p>
<h3 id="putval">putVal方法</h3>
<p>put方法接受两个参数，分别是key和value。key用于计算value存放位置。具体逻辑如下：</p>
<ul>
<li>首先根据hash方法得到key的hash值，这个值是key的hashcode^key的hashcode右移16位的结果</li>
<li>然后调用putVal方法。putVal方法是最终操作table的方法，不管是put还是putIfAbsent或者是putAll，最终都由putVal完成将值存放到table里。<ul>
<li>如果table为空或者table的长度为0，通过resize方法初始化</li>
<li>接下来是存放值的操作。通过hashcode按位与table的length-1，得到数据存放的桶。如果桶里为空，则构造一个Node存放到table中前面通过按位与所得到的索引位置。说明一下：Node实际上是Map.Entry的一个实现。在1.8中Map.Entry在HashMap里有2个实现类。Node是其中之一，它是一个单向链表结构。另一个是TreeNode，TreeNode是红黑树结构。如果hash冲突的元素在2-8个那么则使用Node这个数据结构来存放元素，如果超过8则使用TreeNode来存放。</li>
<li>hash冲突分3种情况处理:<ul>
<li>第一种是重复设置，使用同一个key去设置多次值也被视为是hash冲突的。判断是否为同一个key的方式：要么同一个对象，也就能够用==判断，要么是key使用equals方法判断返回true。这种情况直接覆盖原来的数据。</li>
<li>第二种情况，就是冲突元素已经大于8个，将数据存到前面提到的TreeNode中</li>
<li>第三种情况，有冲突，但不大于8个时。存放到单向链表Node中。遍历单向链表：用了一个死循环，退出条件为当前桶的最后一个元素的next为null或者找到了重复的key。当最后一个元素为null时则认为当前put的元素是一个新的元素，将他放到链表的末端。当key重复了什么也不做，跳出循环后再处理。跳出循环后如果检测到变量e不为null，则认为当前这个key是重复的，然后重新赋值</li>
</ul>
</li>
</ul>
</li>
</ul>
<p>putVal方法的代码如下：</p>
<pre><code class="java">final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node&lt;K,V&gt;[] tab; Node&lt;K,V&gt; p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)//如果table为null或者table的长度为0
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) &amp; hash]) == null)//没有hash冲突，构造一个Node并存放到对应的位置
            tab[i] = newNode(hash, key, value, null);
        else {
            Node&lt;K,V&gt; e; K k;
            if (p.hash == hash &amp;&amp;
                ((k = p.key) == key || (key != null &amp;&amp; key.equals(k)))) //p代表当前桶的第一关元素。此处判断的是地一个元素的key是否与新存入的相同，如果相同就将p赋值给e，待后面重新赋值
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode&lt;K,V&gt;)p).putTreeVal(this, tab, hash, key, value);//将元素put到TreeNode
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {//如果循环到最后一个都没有相同的key，则将存入的节点追加到链表最后一个元素的next
                        p.next = newNode(hash, key, value, null);
                        if (binCount &gt;= TREEIFY_THRESHOLD - 1) // -1 for 1st 
                            treeifyBin(tab, hash);//转换为TreeNode
                        break;
                    }
                    if (e.hash == hash &amp;&amp;
                        ((k = e.key) == key || (key != null &amp;&amp; key.equals(k)))) //链表中已经存在相同的key，处理手法与上面的第一个if类似，e的赋值操作由for循环后的if完成
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size &gt; threshold)//是否重置桶
            resize();
        afterNodeInsertion(evict);
        return null;
    }
</code></pre>

<p>测试代码如下</p>
<pre><code class="java">public class Pojo {
    public int hashCode() {
        return 0xff;
    }
    public static void main(String... args){
        Map&lt;Pojo,Integer&gt; map = new HashMap&lt;Pojo,Integer&gt;();
        for (int i = 0; i&lt; 10 ;) {
            Pojo p = new Pojo();
            map.put(p,++i);
        }
    }
}
</code></pre>

<h3 id="loadfactorinitialcapacity">构造参数loadFactor与initialCapacity</h3>
<ul>
<li>loadFactor 默认0.75。</li>
<li>initialCapacity 默认16。tableSizeFor这个方法会根据实际传入的值进行再一次校验。用|=进行1,2,4,8,16的右移位后再确定threshold大小，如果小于0,则为1,如果大于1&lt;&lt;30则为1&lt;&lt;30，否则为initialCapacity+1</li>
</ul>