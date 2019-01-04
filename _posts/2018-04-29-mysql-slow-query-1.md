---
layout: post
title:     优化器选择索引出错踩的坑
date:    2018-04-29 12:20:11
tags:
- mysql
---

<h1><a id="_0"></a>背景</h1>
<p>线上一条sql按正常执行计划应该是毫秒级出结果都。突然成了慢查询。dba反馈此sql执行了3000秒还未返回结果。sql做了‘脱敏’。简单描述下：m_ext_value是存放数据表，它是一张达40亿行的大表，ext_attr是定义表，只有几百行记录。通过这种方式，将原来按列存储的数据变成按行存储。</p>
<pre><code class="language-sql"><span class="hljs-operator"><span class="hljs-keyword">select</span> ext.m_id <span class="hljs-keyword">as</span> <span class="hljs-keyword">mId</span>, <span class="hljs-keyword">attr</span>.<span class="hljs-keyword">id</span> <span class="hljs-keyword">as</span> attrId, <span class="hljs-keyword">attr</span>.code <span class="hljs-keyword">as</span> attrCode, <span class="hljs-keyword">attr</span>.data_type <span class="hljs-keyword">as</span> dataType,
ext.value_datetime <span class="hljs-keyword">as</span> attrValueDatetime, ext.value_decimal <span class="hljs-keyword">as</span> attrValueDecimal, ext.value_int <span class="hljs-keyword">as</span> attrValueInt, 
ext.value_l_varchar <span class="hljs-keyword">as</span> attrValueLVarchar, ext.value_s_varchar <span class="hljs-keyword">as</span> attrValueSVarchar <span class="hljs-keyword">from</span> m_ext_value ext
<span class="hljs-keyword">inner</span> <span class="hljs-keyword">join</span>  ext_attr <span class="hljs-keyword">attr</span>
<span class="hljs-keyword">on</span> ext.entity_ext_id = <span class="hljs-keyword">attr</span>.<span class="hljs-keyword">id</span> <span class="hljs-keyword">and</span> <span class="hljs-keyword">attr</span>.entity_type = <span class="hljs-number">2</span> <span class="hljs-keyword">and</span> <span class="hljs-keyword">attr</span>.is_deleted = <span class="hljs-number">0</span> 
<span class="hljs-keyword">and</span> ext.is_deleted = <span class="hljs-number">0</span> <span class="hljs-keyword">and</span> <span class="hljs-keyword">attr</span>.<span class="hljs-keyword">status</span> = <span class="hljs-number">1</span> 
<span class="hljs-keyword">where</span> ext.m_id <span class="hljs-keyword">in</span> ( <span class="hljs-number">446351894</span> ) 
<span class="hljs-keyword">order</span> <span class="hljs-keyword">by</span> ext.<span class="hljs-keyword">id</span> <span class="hljs-keyword">desc</span>
</span></code></pre>
<h1><a id="_13"></a>分析</h1>
<p>通过explain 上面语句得到等结果是</p>
<table class="table table-striped table-bordered">
<thead>
<tr>
<th style="text-align:left">ID</th>
<th style="text-align:left">读取类型</th>
<th style="text-align:left">读取表</th>
<th style="text-align:left">连接类型</th>
<th style="text-align:left">可用索引</th>
<th style="text-align:left">使用索引</th>
<th style="text-align:left">索引长度</th>
<th style="text-align:left">额外引用</th>
<th style="text-align:left">扫描行数</th>
<th style="text-align:left">额外描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">1</td>
<td style="text-align:left">SIMPLE</td>
<td style="text-align:left">ext</td>
<td style="text-align:left">ref</td>
<td style="text-align:left">idx_m_id,idx_ext_attr_id</td>
<td style="text-align:left">idx_m_id</td>
<td style="text-align:left">8</td>
<td style="text-align:left">const</td>
<td style="text-align:left">9</td>
<td style="text-align:left">Using where</td>
</tr>
<tr>
<td style="text-align:left">1</td>
<td style="text-align:left">SIMPLE</td>
<td style="text-align:left">attr</td>
<td style="text-align:left">eq_ref</td>
<td style="text-align:left">PRIMARY,uni_ext_attr</td>
<td style="text-align:left">PRIMARY</td>
<td style="text-align:left">8</td>
<td style="text-align:left">db1.ext.ext_attr_id</td>
<td style="text-align:left">1</td>
<td style="text-align:left">Using where</td>
</tr>
</tbody>
</table>
<p>通过这个执行计划来看，mysql走的索引是正确的，而且过滤性很好。然而线上执行缺有问题，为啥3000s还没返回？<br>
然后dba对当前这次查询进行了explain，发现它使用的执行计划和上面的不一样。</p>
<blockquote>
<p>explain for connection 12345;</p>
</blockquote>
<table class="table table-striped table-bordered">
<thead>
<tr>
<th style="text-align:left">ID</th>
<th style="text-align:left">读取类型</th>
<th style="text-align:left">读取表</th>
<th style="text-align:left">连接类型</th>
<th style="text-align:left">可用索引</th>
<th style="text-align:left">使用索引</th>
<th style="text-align:left">索引长度</th>
<th style="text-align:left">额外引用</th>
<th style="text-align:left">扫描行数</th>
<th style="text-align:left">额外描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left">1</td>
<td style="text-align:left">SIMPLE</td>
<td style="text-align:left">attr</td>
<td style="text-align:left">ref</td>
<td style="text-align:left">PRIMARY,uni_ext_attr</td>
<td style="text-align:left">uni_ext_attr</td>
<td style="text-align:left">1</td>
<td style="text-align:left">const</td>
<td style="text-align:left">44</td>
<td style="text-align:left">Using where; Using temporary; Using filesort</td>
</tr>
<tr>
<td style="text-align:left">1</td>
<td style="text-align:left">SIMPLE</td>
<td style="text-align:left">ext</td>
<td style="text-align:left">ref</td>
<td style="text-align:left">idx_m_id,idx_ext_attr_id</td>
<td style="text-align:left">idx_ext_attr_id</td>
<td style="text-align:left">8</td>
<td style="text-align:left"><a href="http://db1.ext.id">db1.ext.id</a></td>
<td style="text-align:left">7344</td>
<td style="text-align:left">Using where</td>
</tr>
</tbody>
</table>
<p>dba给的解释是：这个时候可能在计算统计信息，导致优化器在这一刻看到的统计信息是错误的，导致优化器认为此索引过滤性不好，就转用小表作为驱动表来join，悲剧就此发生。通过上述表格可以看到扫描行数已经去到7344，实际只有9左右。</p>
<p>join的本质可以粗略理解为两个循环嵌套。外循环称作<strong>驱动表</strong>。此处可以理解为，将小表作为外循环，根据<strong><a href="http://attr.id">attr.id</a></strong>与大表进行关联。意思就是要把大表这几十亿数据轮几十遍。我的个乖乖，没挂就算幸运了。</p>
<h1><a id="_33"></a>破解</h1>
<p>将<em>inner join</em>改为<em>stiraight_join</em>可避免此问题。让大表作为驱动表，强制其使用m_id列的索引（扫出来的结果集很小），然后对小表进行join。</p>
<blockquote>
<p>mysql决策谁做驱动表由查询的数据集大小确定，<strong>谁的数据集小谁做驱动表</strong></p>
</blockquote>
<h1><a id="_38"></a>其他</h1>
<h2><a id="_39"></a>设计</h2>
<p>把列变成行这种做法非常不好。数据量急剧膨胀无法控制，拆库都必须依赖主表。我觉得还是建另一个表来存放好一点。开发友好，理解也简单，查问题方便。所以表设计时一定要注意。一定要对自己对业务表有一个比较准确的预估，以及扩展计划。</p>
<h2><a id="straight_join_42"></a>straight_join</h2>
<p>straight_join其实不是什么特别join，他会干扰mysql对表的连接顺序。强制join前的表作为驱动表（外循环）</p>
<h2><a id="_45"></a>…</h2>
<p>虽然mysql加入了bka、mrr、icp等。但其实并不能完全解决性能问题，特别是io的坑。建议不要在生产上使用join，尽量简化查询。<strong>根据简单的key来查不怕查多次，就怕慢查询。</strong></p>
<p>另外，随着业务的发展后续要想拆库，需要把原来用join方式联合查询的表分到两个不同的库，那就改动大了。随着微服务的兴起，这是很常见的事。</p>
