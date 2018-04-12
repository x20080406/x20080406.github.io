---
layout: post
title:     mysql范围连续/缺失问题
date:    2018-04-13 01:11:11
tags:
- mysql
---

<h1><a id="_0"></a>题目(不知道是哪里的面试题)</h1>
<p>x市间了一个新的体育馆，每日人均流量信息被记录在这3列信息中：序号（id，自增），日期（date），人流量（people）。<br>
请编写一个sql，找出高峰期时段，要求连续3天及以上，并且每天人流量均不少于100.</p>
<p>这话有点拗口，写程序的10有八九不能清楚的用文字表达脑子想什么。<br>
简单翻译一下。用sql找出连续3天人流量都超过100都记录。</p>
<p>例如：表<strong>stadium</strong></p>
<pre><code>+-------+----------+--------+
|1      |2017-01-01|10      |
|2      |2017-01-02|109     |
|3      |2017-01-03|102     |
|4      |2017-01-04|70      |
|5      |2017-01-05|140     |
|6      |2017-01-06|1012    |
|7      |2017-01-07|123     |
|8      |2017-01-08|101     |
+-------+----------+--------+
</code></pre>
<h1><a id="_21"></a>解法</h1>
<p>目前发现两种解法<br>
法1比较土</p>
<pre><code class="language-sql"><span class="hljs-operator"><span class="hljs-keyword">select</span> * <span class="hljs-keyword">from</span> stadium s
 <span class="hljs-keyword">where</span> <span class="hljs-keyword">exists</span> (
        <span class="hljs-keyword">select</span> <span class="hljs-number">1</span> <span class="hljs-keyword">from</span> stadium s1 <span class="hljs-keyword">where</span> s1.<span class="hljs-keyword">id</span> <span class="hljs-keyword">in</span> (s.<span class="hljs-keyword">id</span> ,s.<span class="hljs-keyword">id</span>+<span class="hljs-number">1</span>,s.<span class="hljs-keyword">id</span>+<span class="hljs-number">2</span>) <span class="hljs-keyword">and</span> s1.people &gt;= <span class="hljs-number">100</span> <span class="hljs-keyword">having</span> <span class="hljs-keyword">count</span>(<span class="hljs-number">1</span>) =<span class="hljs-number">3</span> )
    <span class="hljs-keyword">OR</span>  <span class="hljs-keyword">exists</span> (
        <span class="hljs-keyword">select</span> <span class="hljs-number">1</span> <span class="hljs-keyword">from</span> stadium s1 <span class="hljs-keyword">where</span> s1.<span class="hljs-keyword">id</span> <span class="hljs-keyword">in</span> (s.<span class="hljs-keyword">id</span>-<span class="hljs-number">1</span> ,s.<span class="hljs-keyword">id</span>,s.<span class="hljs-keyword">id</span>+<span class="hljs-number">1</span>) <span class="hljs-keyword">and</span> s1.people &gt;= <span class="hljs-number">100</span> <span class="hljs-keyword">having</span> <span class="hljs-keyword">count</span>(<span class="hljs-number">1</span>) =<span class="hljs-number">3</span> )
    <span class="hljs-keyword">OR</span>  <span class="hljs-keyword">exists</span> (
        <span class="hljs-keyword">select</span> <span class="hljs-number">1</span> <span class="hljs-keyword">from</span> stadium s1 <span class="hljs-keyword">where</span> s1.<span class="hljs-keyword">id</span> <span class="hljs-keyword">in</span> (s.<span class="hljs-keyword">id</span>-<span class="hljs-number">2</span> ,s.<span class="hljs-keyword">id</span>-<span class="hljs-number">1</span>,s.<span class="hljs-keyword">id</span>) <span class="hljs-keyword">and</span> s1.people &gt;= <span class="hljs-number">100</span> <span class="hljs-keyword">having</span> <span class="hljs-keyword">count</span>(<span class="hljs-number">1</span>) =<span class="hljs-number">3</span> );</span>

</code></pre>
<p>法2，经典求范围连续/范围缺失问题的sql</p>
<pre><code class="language-sql"><span class="hljs-operator"><span class="hljs-keyword">select</span> <span class="hljs-keyword">m</span>.cq,<span class="hljs-keyword">f</span>.bq,<span class="hljs-keyword">m</span>.<span class="hljs-keyword">id</span>, <span class="hljs-keyword">m</span>.<span class="hljs-built_in">date</span>, <span class="hljs-keyword">m</span>.people <span class="hljs-keyword">from</span> (
    <span class="hljs-keyword">select</span> @rw:=<span class="hljs-keyword">ifnull</span>(@rw,<span class="hljs-number">0</span>)+<span class="hljs-number">1</span> <span class="hljs-keyword">as</span> rw, <span class="hljs-keyword">id</span>-@rw <span class="hljs-keyword">as</span> cq, <span class="hljs-keyword">id</span>, <span class="hljs-built_in">date</span>, people <span class="hljs-keyword">from</span>  stadium a ,(<span class="hljs-keyword">select</span> @rw:=<span class="hljs-number">0</span>) b <span class="hljs-keyword">where</span> a.people &gt; <span class="hljs-number">100</span>
) <span class="hljs-keyword">m</span>, (
    <span class="hljs-keyword">select</span> a.bq,<span class="hljs-keyword">count</span>(*) <span class="hljs-keyword">as</span> cnt <span class="hljs-keyword">from</span>  (
        <span class="hljs-keyword">select</span> @rn:=<span class="hljs-keyword">ifnull</span>(@rn,<span class="hljs-number">0</span>)+<span class="hljs-number">1</span> <span class="hljs-keyword">as</span> rn, <span class="hljs-keyword">id</span>-@rn <span class="hljs-keyword">as</span> bq,<span class="hljs-keyword">id</span>, <span class="hljs-built_in">date</span>, people <span class="hljs-keyword">from</span>  stadium a,
        (<span class="hljs-keyword">select</span> @rn:=<span class="hljs-number">0</span>) r <span class="hljs-keyword">where</span> a.people &gt; <span class="hljs-number">100</span>
    ) a
    <span class="hljs-keyword">group</span> <span class="hljs-keyword">by</span> a.bq <span class="hljs-keyword">having</span> cnt &gt; <span class="hljs-number">2</span>
) <span class="hljs-keyword">f</span> <span class="hljs-keyword">where</span> <span class="hljs-keyword">m</span>.cq = <span class="hljs-keyword">f</span>.bq;</span>


</code></pre>
<h1><a id="mysqldump_50"></a>mysqldump出的数据</h1>
<pre><code class="language-shell">mysqldump -uroot -p db1 stadium
</code></pre>
<pre><code class="language-sql"><span class="hljs-comment">-- MySQL dump 10.13  Distrib 5.7.21, for macos10.13 (x86_64)</span>
<span class="hljs-comment">--</span>
<span class="hljs-comment">-- Host: localhost    Database: db1</span>
<span class="hljs-comment">-- ------------------------------------------------------</span>
<span class="hljs-comment">-- Server version   5.7.21-log</span>

<span class="hljs-comment">/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */</span>;
<span class="hljs-comment">/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */</span>;
<span class="hljs-comment">/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */</span>;
<span class="hljs-comment">/*!40101 SET NAMES utf8 */</span>;
<span class="hljs-comment">/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */</span>;
<span class="hljs-comment">/*!40103 SET TIME_ZONE='+00:00' */</span>;
<span class="hljs-comment">/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */</span>;
<span class="hljs-comment">/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */</span>;
<span class="hljs-comment">/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */</span>;
<span class="hljs-comment">/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */</span>;

<span class="hljs-comment">--</span>
<span class="hljs-comment">-- Table structure for table `stadium`</span>
<span class="hljs-comment">--</span>

<span class="hljs-operator"><span class="hljs-keyword">DROP</span> <span class="hljs-keyword">TABLE</span> <span class="hljs-keyword">IF</span> <span class="hljs-keyword">EXISTS</span> <span class="hljs-string">`stadium`</span>;</span>
<span class="hljs-comment">/*!40101 SET @saved_cs_client     = @@character_set_client */</span>;
<span class="hljs-comment">/*!40101 SET character_set_client = utf8 */</span>;
<span class="hljs-operator"><span class="hljs-keyword">CREATE</span> <span class="hljs-keyword">TABLE</span> <span class="hljs-string">`stadium`</span> (
  <span class="hljs-string">`id`</span> <span class="hljs-built_in">int</span>(<span class="hljs-number">11</span>) <span class="hljs-keyword">DEFAULT</span> <span class="hljs-literal">NULL</span>,
  <span class="hljs-string">`date`</span> <span class="hljs-keyword">timestamp</span> <span class="hljs-literal">NULL</span> <span class="hljs-keyword">DEFAULT</span> <span class="hljs-literal">NULL</span>,
  <span class="hljs-string">`people`</span> <span class="hljs-built_in">int</span>(<span class="hljs-number">11</span>) <span class="hljs-keyword">DEFAULT</span> <span class="hljs-literal">NULL</span>
) <span class="hljs-keyword">ENGINE</span>=<span class="hljs-keyword">InnoDB</span> <span class="hljs-keyword">DEFAULT</span> <span class="hljs-keyword">CHARSET</span>=utf8mb4;</span>
<span class="hljs-comment">/*!40101 SET character_set_client = @saved_cs_client */</span>;

<span class="hljs-comment">--</span>
<span class="hljs-comment">-- Dumping data for table `stadium`</span>
<span class="hljs-comment">--</span>

<span class="hljs-operator"><span class="hljs-keyword">LOCK</span> <span class="hljs-keyword">TABLES</span> <span class="hljs-string">`stadium`</span> WRITE;</span>
<span class="hljs-comment">/*!40000 ALTER TABLE `stadium` DISABLE KEYS */</span>;
<span class="hljs-operator"><span class="hljs-keyword">INSERT</span> <span class="hljs-keyword">INTO</span> <span class="hljs-string">`stadium`</span> <span class="hljs-keyword">VALUES</span> (<span class="hljs-number">1</span>,<span class="hljs-string">'2016-12-31 16:00:00'</span>,<span class="hljs-number">101</span>),(<span class="hljs-number">2</span>,<span class="hljs-string">'2017-01-01 16:00:00'</span>,<span class="hljs-number">109</span>),(<span class="hljs-number">3</span>,<span class="hljs-string">'2017-01-02 16:00:00'</span>,<span class="hljs-number">159</span>),(<span class="hljs-number">4</span>,<span class="hljs-string">'2017-01-03 16:00:00'</span>,<span class="hljs-number">99</span>),(<span class="hljs-number">5</span>,<span class="hljs-string">'2017-01-04 16:00:00'</span>,<span class="hljs-number">149</span>),(<span class="hljs-number">6</span>,<span class="hljs-string">'2017-01-05 16:00:00'</span>,<span class="hljs-number">1455</span>),(<span class="hljs-number">7</span>,<span class="hljs-string">'2017-01-06 16:00:00'</span>,<span class="hljs-number">199</span>),(<span class="hljs-number">8</span>,<span class="hljs-string">'2017-01-07 16:00:00'</span>,<span class="hljs-number">192</span>);</span>
<span class="hljs-comment">/*!40000 ALTER TABLE `stadium` ENABLE KEYS */</span>;
<span class="hljs-operator"><span class="hljs-keyword">UNLOCK</span> <span class="hljs-keyword">TABLES</span>;</span>
<span class="hljs-comment">/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */</span>;

<span class="hljs-comment">/*!40101 SET SQL_MODE=@OLD_SQL_MODE */</span>;
<span class="hljs-comment">/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */</span>;
<span class="hljs-comment">/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */</span>;
<span class="hljs-comment">/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */</span>;
<span class="hljs-comment">/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */</span>;
<span class="hljs-comment">/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */</span>;
<span class="hljs-comment">/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */</span>;

<span class="hljs-comment">-- Dump completed on 2018-04-13  1:01:32</span>


</code></pre>
