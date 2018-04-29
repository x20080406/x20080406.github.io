---
layout: post
title:     优化器选择索引出错踩的坑
date:    2018-04-29 12:20:11
tags:
- mysql
---

<h1 id="h1-u80CCu666F"><a name="背景" class="reference-link"></a><span class="header-link octicon octicon-link"></span>背景</h1><p>线上一条sql按正常执行计划应该是毫秒级出结果都。突然成了慢查询。dba反馈此sql执行了3000秒还未返回结果。sql做了‘脱敏’。简单描述下：m_ext_value是存放数据表，它是一张达40亿行的大表，ext_attr是定义表，只有几百行记录。</p>
<pre><code class="lang-sql">select ext.m_id as mId, attr.id as attrId, attr.code as attrCode, attr.data_type as dataType,
ext.value_datetime as attrValueDatetime, ext.value_decimal as attrValueDecimal, ext.value_int as attrValueInt, 
ext.value_l_varchar as attrValueLVarchar, ext.value_s_varchar as attrValueSVarchar from m_ext_value ext
inner join  ext_attr attr
on ext.entity_ext_id = attr.id and attr.entity_type = 2 and attr.is_deleted = 0 
and ext.is_deleted = 0 and attr.status = 1 
where ext.m_id in ( 446351894 ) 
order by ext.id desc
</code></pre>
<h1 id="h1-u5206u6790"><a name="分析" class="reference-link"></a><span class="header-link octicon octicon-link"></span>分析</h1><p>通过explain 上面语句得到等结果是<br>| ID | 读取类型   | 读取表  | 连接类型   | 可用索引                            | 使用索引     | 索引长度 | 额外引用                             | 扫描行数 | 额外描述        |<br>|:—-|:———-|:——-|:———-|:————————————————|:————-|:——-|:————————————————-|:——-|:——————|<br>| 1  | SIMPLE | ext  | ref    | idx_m_id,idx_ext_attr_id | idx_m_id | 8    | const                            | 9    | Using where |<br>| 1  | SIMPLE | attr | eq_ref | PRIMARY,uni_ext_attr     | PRIMARY  | 8    | db1.ext.ext_attr_id | 1    | Using where |
<p>通过这个执行计划来看，mysql走的索引是正确的，而且过滤性很好。然而线上执行缺有问题，为啥3000s还没返回？<br>然后dba对当前这次查询进行了explain，发现它使用的执行计划和上面的不一样。
<blockquote>
<p>explain for connection 12345;</p>
</blockquote>
<table>
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
<td style="text-align:left">db1.ext.id</td>
<td style="text-align:left">7344</td>
<td style="text-align:left">Using where</td>
</tr>
</tbody>
</table>
<p>dba给的解释是：这个时候可能在计算统计信息，导致优化器在这一刻看到的统计信息是错误的，导致优化器认为此索引过滤性不好，就转用小表作为驱动表来join，悲剧就此发生。通过上述表格可以看到cardinality已经去到7344。</p>
<p>join的本质可以粗略理解未两个嵌套循环。外循环称作驱动表。此处可以理解为，将小表作为外循环，根据<strong>attr.id</strong>与大表进行关联。意思就是要把大表这几十亿数据轮几十遍。我的给乖乖，没挂就算幸运来。</p>
<h1 id="h1-u7834u89E3"><a name="破解" class="reference-link"></a><span class="header-link octicon octicon-link"></span>破解</h1><p>将<em>inner join</em>改为<em>stiraight_join</em>可避免此问题。让大表作为驱动表，强制其扫描时m_id列的索引，然后对小表进行join</p>
<h1 id="h1-u5176u4ED6"><a name="其他" class="reference-link"></a><span class="header-link octicon octicon-link"></span>其他</h1><h2 id="h2-u8BBEu8BA1"><a name="设计" class="reference-link"></a><span class="header-link octicon octicon-link"></span>设计</h2><p>把列变成行这种做法非常不好。数据量急剧膨胀就悲剧来。我觉得还是建另一个表来存放好一点。开发友好，理解也简单，查问题方便。所以表设计时一定要注意。一定要对自己对业务表有一个比较准确的预估，以及扩展计划。</p>
<h2 id="h2-stiraight_join"><a name="stiraight_join" class="reference-link"></a><span class="header-link octicon octicon-link"></span>stiraight_join</h2><p>straight_join其实不是什么特别join，他会干扰mysql对表的连接顺序。强制join前的表作为驱动表（外循环）</p>
<h2 id="h2-join"><a name="join" class="reference-link"></a><span class="header-link octicon octicon-link"></span>join</h2><p>join本身其实就是一个双重循环。虽然mysql加入了mrr、icp等，但其实并不能完全解决性能问题，特别是io的坑。建议不要在生产上使用join，精良简化查询。<strong>根据简单的key来不怕查多次，就怕慢查询。</strong></p>
