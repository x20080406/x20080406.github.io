---
layout: post
title:     mysql范围连续/缺失问题
date:    2018-04-13 01:11:11
tags:
- mysql
---

# 题目(不知道是哪里的面试题)
x市间了一个新的体育馆，每日人均流量信息被记录在这3列信息中：序号（id，自增），日期（date），人流量（people）。
请编写一个sql，找出高峰期时段，要求连续3天及以上，并且每天人流量均不少于100.

这话有点拗口，写程序的10有八九不能清楚的用文字表达脑子想什么。
简单翻译一下。用sql找出连续3天人流量都超过100都记录。

例如：表**stadium**
```
+-------+----------+--------+
|1      |2017-01-01|10      |
|2      |2017-01-02|109     |
|3      |2017-01-03|102     |
|4      |2017-01-04|70      |
|5      |2017-01-05|140     |
|6      |2017-01-06|1012    |
|7      |2017-01-07|123     |
|8      |2017-01-08|101     |
+-------+----------+--------+
```

# 解法
目前发现两种解法
法1比较土
```sql
select * from stadium s
 where exists (
 		select 1 from stadium s1 where s1.id in (s.id ,s.id+1,s.id+2) and s1.people >= 100 having count(1) =3 )
	OR  exists (
		select 1 from stadium s1 where s1.id in (s.id-1 ,s.id,s.id+1) and s1.people >= 100 having count(1) =3 )
	OR  exists (
		select 1 from stadium s1 where s1.id in (s.id-2 ,s.id-1,s.id) and s1.people >= 100 having count(1) =3 );

```

法2，经典求范围连续/范围缺失问题的sql

如果没有少数据，则id减行号为0。每断几条，则id减行号的数量会增大几个。通过这个特性将连续的数据通过（id-行号）组织在一起。最后通过id-行号作为分组进行最后的统计，然后关联一把就得到要的数据。
```sql
select m.cq,f.bq,m.id, m.date, m.people from (
	select @rw:=ifnull(@rw,0)+1 as rw, id-@rw as cq, id, date, people from  stadium a ,(select @rw:=0) b where a.people > 100
) m, (
	select a.bq,count(*) as cnt from  (
		select @rn:=ifnull(@rn,0)+1 as rn, id-@rn as bq,id, date, people from  stadium a,
		(select @rn:=0) r where a.people > 100
	) a
	group by a.bq having cnt > 2
) f where m.cq = f.bq;


```

# mysqldump出的数据
```shell
mysqldump -uroot -p db1 stadium
```

```sql
-- MySQL dump 10.13  Distrib 5.7.21, for macos10.13 (x86_64)
--
-- Host: localhost    Database: db1
-- ------------------------------------------------------
-- Server version	5.7.21-log

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `stadium`
--

DROP TABLE IF EXISTS `stadium`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `stadium` (
  `id` int(11) DEFAULT NULL,
  `date` timestamp NULL DEFAULT NULL,
  `people` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `stadium`
--

LOCK TABLES `stadium` WRITE;
/*!40000 ALTER TABLE `stadium` DISABLE KEYS */;
INSERT INTO `stadium` VALUES (1,'2016-12-31 16:00:00',101),(2,'2017-01-01 16:00:00',109),(3,'2017-01-02 16:00:00',159),(4,'2017-01-03 16:00:00',99),(5,'2017-01-04 16:00:00',149),(6,'2017-01-05 16:00:00',1455),(7,'2017-01-06 16:00:00',199),(8,'2017-01-07 16:00:00',192);
/*!40000 ALTER TABLE `stadium` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2018-04-13  1:01:32


```

