---
layout: post
title:  mysql的batch操作支持
date:   2016-04-10 13:27:11
tags:
 - java
 - database
---

<h1>问题</h1>
<p>mysql执行20000条数据插入，时间耗掉约70秒。使用的是预编译执行。</p>

<h1>分析</h1>
<p>将下列参数加入到jdbcurl中然后分析日志。通过mysql的logger将日志打印出来发现，addBatch/executeBatch无效。实际上还是逐条执行的。
</p>
<pre><code>logger=Slf4JLogger&amp;profileSQL=true</code></pre>
<p>日志截取：
</p>
<pre><code>INFO  2016-04-10 13:39:58,534 pool-1-thread-26 MySQL[85]:
QUERY created: Sun Apr 10 13:39:58 CST 2016 duration: 77 connection: 1132 statement: 51 resultset: 0 message: insert into `Bar` (name,m_item_key) values ('pool-1-thread-2617487',17487)

INFO  2016-04-10 13:39:58,534 pool-1-thread-26 MySQL[85]:
FETCH created: Sun Apr 10 13:39:58 CST 2016 duration: 0 connection: 1132 statement: 51 resultset: 0

INFO  2016-04-10 13:39:58,535 pool-1-thread-28 MySQL[85]:
QUERY created: Sun Apr 10 13:39:58 CST 2016 duration: 77 connection: 1131 statement: 50 resultset: 0 message: insert into `Bar` (name,m_item_key) values ('pool-1-thread-2817287',17287)

INFO  2016-04-10 13:39:58,535 pool-1-thread-28 MySQL[85]:
FETCH created: Sun Apr 10 13:39:58 CST 2016 duration: 0 connection: 1131 statement: 50 resultset: 0

INFO  2016-04-10 13:39:58,536 pool-1-thread-10 MySQL[85]:
QUERY created: Sun Apr 10 13:39:58 CST 2016 duration: 78 connection: 1122 statement: 60 resultset: 0 message: insert into `Bar` (name,m_item_key) values ('pool-1-thread-1019087',19087)</code></pre>

<h1>处理办法</h1>
<p>查阅官方文档发现通过设置rewriteBatchedStatements可以使addBatch/executeBatch生效。设置后日志截取：
</p>
<pre><code>INFO  2016-04-10 13:02:00,200 pool-1-thread-23 MySQL[85]:
QUERY created: Sun Apr 10 13:02:00 CST 2016 duration: 48 connection: 1102 statement: 257 resultset: 0 message: insert into `Bar` (name,m_item_key) values ('pool-1-thread-239800',9800),('pool-1-thread-239799',9799),('pool-1-thread-239798',9798),('pool-1-thread-239797',9797),('pool-1-thread-239796',9796),('pool-1-thread-239795',9795),('pool-1-thread-239794',9794),('pool-1-thread-239793',9793),('pool-1-thread-239792',9792),('pool-1-thread-239791',9791),('pool-1-thread-239790',9790),('pool-1-thread-239789',9789),('pool-1-thread-239788',9788),('pool-1-thread-239787',9787),('pool-1-thread-239786',9786),('pool-1-thread-239785',9785),('pool-1-thread-239784',9784),('pool-1-thread-239783',9783),('pool-1-thread-239782',9782),('pool-1-thread-239781',9781),('pool-1-thread-239780',9780),('pool-1-thread-239779',9779),('pool-1-thread-239778',9778),('pool-1-thread-239777',9777),('pool-1-thread-239776',9776),('pool-1-thread-239775',9775),('pool-1-thread-239774',9774),('pool-1-thread-239773',9773),('pool-1-thread-239772',9772),('pool-1-thread-239771',9771),('pool-1-thread-239770',9770),('pool-1-thread-239769',9769),('pool-1-thread-239768',9768),('pool-1-thread-239767',9767),('pool-1-thread-239766',9766),('pool-1-thread-239765',9765),('pool-1-thread-239764',9764),('pool-1-thread-239763',9763),('pool-1-thread-239762',9762),('pool-1-thread-239761',9761),('pool-1-thread-239760',9760),('pool-1-thread-239759',9759),('pool-1-thread-239758',9758),('pool-1-thread-239757',9757),('pool-1-thread-239756',9756),('pool-1-thread-239755',9755),('pool-1-thread-239754',9754),('pool-1-thread-239753',9753),('pool-1-thread-239752',9752),('pool-1-thread-239751',9751),('pool-1-thread-239750',9750),('pool-1-thread-239749',9749),('pool-1-thread-239748',9748),('pool-1-thread-239747',9747),('pool-1-thread-239746',9746),('pool-1-thread-239745',9745),('pool-1-thread-239744',9744),('pool-1-thread-239743',9743),('pool-1-thread-239742',9742),('pool-1-thread-239741',9741),('pool-1-thread-239740',9740),('pool-1-thread-239739',9739),('pool-1-thread-239738',9738),('pool-1-thread-239737',9737),('pool-1-thread-239736',9736),('pool-1-thread-239735',9735),('pool-1-thread-239734',9 ... (truncated)</code></pre>

<h1>结论</h1>
<p>mysql-connector-java（5.1.31）驱动默认没有将rewriteBatchedStatements设置为true，将它设置为true后可以支持批量处理。
</p>
<p>最终jdbcurl如下：
</p>
<pre><code>jdbc:mysql://127.0.0.1/smart?allowMultiQueries=true&amp;rewriteBatchedStatements=true&amp;logger=Slf4JLogger&amp;profileSQL=true</code></pre>
<h1>参考网站</h1>
<p><a href="https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html">https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html</a></p>
