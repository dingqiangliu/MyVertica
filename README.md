<html lang="zn_CN"> <head> <meta charset='utf-8'> <title>MyVertica 中文社区常见问题</title> </head> <body>

MyVertica中文社区说明
==========
这里是Vertica爱好者关于Vertica相关技术交流、学习和互助的社区，不负责Vertica售后支持。

售后支持关于售后支持: 对于Vertica正式客户，建议中文邮件通过  china_vertica_support@hpe.com 咨询（verticahelp@hpe.com只接受英文邮件），他们是专职的中文支持团队，响应会更即时。

 - china_vertica_support@hpe.com: 9x5, Chinese
 - verticahelp@hpe.com: 7x24 for production issues, English
 - production critical issue support: 1-877-669-1295 or 1-617-386-4500, English. 


相关连接
----------
 * [本文档(https://github.com/dingqiangliu/MyVertica/blob/master/README.md)](https://github.com/dingqiangliu/MyVertica/blob/master/README.md)
 * [Vertica资源下载(http://pan.baidu.com/s/1jXnMu#path=/vertica)](http://pan.baidu.com/s/1jXnMu#path=/vertica)

常见问题: 如何实现Oracle connectby相似功能？
==========
Connectby 实质上就是表自(外)关联和列转行。如果层级不多，是可以用标准SQL来改写。

其实用Vertica的SDK来实现相同功能的自定义函数，也不太难。参见：[Connectby for Veritca实现(http://pan.baidu.com/s/1jXnMu#path=/vertica/UDx/)](http://pan.baidu.com/s/1jXnMu#path=/vertica/UDx/) 下的 connectby-2013.08.30.tgz

对于Oracle中如下语句的执行效果:
<code><pre>
select id, parent_id, name
  , LEVEL as path_level
  , CONNECT_BY_ROOT id AS id_root
  , SYS_CONNECT_BY_PATH(id, '|') AS id_path
  , CONNECT_BY_ROOT name AS name_root
  , SYS_CONNECT_BY_PATH(name, '|') AS name_path
from company c
  CONNECT BY PRIOR c.id = c.parent_id 
  START WITH c.parent_id = 0
order by id;

 ID PARENT_ID NAME	 PATH_LEVEL ID_ROOT ID_PATH	    NAME_ROOT  NAME_PATH
-----+-----------------+-----+----------+-----------+------------------------
  1	    0 Patrick1		  1	  1 |1		    Patrick1   |Patrick1
  2	    0 Patrick2		  1	  2 |2		    Patrick2   |Patrick2
 12	    1 Jim1			  2	  1 |1|12	    Patrick1   |Patrick1|Jim1
 13	    1 Sandy1		  2	  1 |1|13	    Patrick1   |Patrick1|Sandy1
 14	   13 Brian1		  3	  1 |1|13|14    Patrick1   |Patrick1|Sandy1|Brian1
 15	   13 Otto1			  3	  1 |1|13|15    Patrick1   |Patrick1|Sandy1|Otto1
 22	    2 Jim2			  2	  2 |2|22	    Patrick2   |Patrick2|Jim2
 23	    2 Sandy2		  2	  2 |2|23	    Patrick2   |Patrick2|Sandy2
 24	   23 Brian2		  3	  2 |2|23|24    Patrick2   |Patrick2|Sandy2|Brian2
 25	   23 Otto2			  3	  2 |2|23|25    Patrick2   |Patrick2|Sandy2|Otto2
10 rows selected.
</code></pre>

* * *

在Vertica中实现的UDF中执行的效果:
<code><pre>
select connectby(parent_id, id, name) over () 
from item;

 level | parent_id | id  |   name   | name_root |       name_path        
-------+-----------+-----+----------+-----------+------------------------
     1 |         0 |   1 | Patrick1 | Patrick1  | Patrick1
     1 |         0 |   2 | Patrick2 | Patrick2  | Patrick2
     2 |         1 |  12 | Jim1     | Patrick1  | Patrick1/Jim1
     2 |         1 |  13 | Sandy1   | Patrick1  | Patrick1/Sandy1
     2 |         2 |  22 | Jim2     | Patrick2  | Patrick2/Jim2
     2 |         2 |  23 | Sandy2   | Patrick2  | Patrick2/Sandy2
     3 |        13 | 131 | Brian1   | Patrick1  | Patrick1/Sandy1/Brian1
     3 |        13 | 132 | Otto1    | Patrick1  | Patrick1/Sandy1/Otto1
     3 |        23 | 231 | Brian2   | Patrick2  | Patrick2/Sandy2/Brian2
     3 |        23 | 232 | Otto2    | Patrick2  | Patrick2/Sandy2/Otto2
(10 rows)
</code></pre>



常见问题: 多大的维度表不适合unsegmented？
==========
假设维度表有5000万行，每行200字节，只是数据就需要约10GB。
如果在关联/分组操作中不能用上Merge Join/Group By Pipe, 数据加上hash 表结构、关联或分组操作等，这样的Hash Join/Group 很有可能需要10+ GB内存。内存容易成为多个这中并发负载场景的瓶颈。
所以通常建议：上千万的维度表，最好按与事实表关联的键值 hash 分布到所有节点上。


案例分享: 一条SQL“无中生有”地生成上亿的测试数据
==========
Vertica 提供时间序列 插值功能，可以用来产生任意范围的数值：

<code><pre>
create table if not exists fact 
as /*+ direct */
select num as pk /* 主键 */, num%1000 as fk /* 构造各种外键或其他信息 */ from (
  SELECT extract(epoch from ts)::int as num FROM (
    SELECT '1970-01-01 00:00:01 +0'::TIMESTAMP AS tm
      UNION
    SELECT '1974-01-02 00:00:00 +0'::TIMESTAMP AS tm 
   ) t0 
   TIMESERIES ts AS '1 second' OVER (ORDER BY tm)
) t1
order by pk
encoded by pk encoding DELTAVAL, fk encoding RLE
segmented by hash(pk) all nodes ksafe
;
</code></pre>


案例分享: 主动监控降低运维风险 —— 王国飞@中移在线
==========
## 背景 ##
任何一个生产环境数据库都要求按照规划的RPT和RTO来设计和实施。

Vertica的ROS存储方式可以保证事务提交时已经写磁盘成功，能确保进程、节点重启或故障时的数据完整性和一致性。

Vertica支持实时数据装载的WOS是暂时(缺省5分钟)驻留在内存中的，尽管在多个节点上保留有副本，但如果发生大范围故障(交换机或多个机柜电源故障)，可能会导致内存中的数据全部丢失。尽管这在系统RTO、RPO设计时候已经考虑，可以从消息流中复原，但还是会给运维临场处理带来麻烦和压力。

## 问题 ##
在一次运维活动中停止数据库前，发现 system 视图中 LGE 和 AHM 与 CE 差距较大，从 epochs 视图上看它们差距时间间隔有好几天，执行 select do_tm_task('moveout' ) 也无法改善。

如果这个时候强制停止数据库、或者发生意外导致多个节点宕机，可能会发生数据丢失问题。为了降低系统风险，需要尽快找到原因，把 WOS 中的数据写到 ROS 中。

## 诊断和解决过程 ##

1. 首先执行下面的查询，找到 WOS 存放了哪些表的数据，结果只有一个表 hanwenfeng.tb_rp_ct_86hl_forcast_source_day
<code><pre>
select schema_name,anchor_table_name,p.projection_name,rowcount,start as start_epoch,"end" as end_epoch
from storage_containers sc, vs_wos_containers wc, projections p
where sc.storage_oid=wc.wosid and sc.projection_name=p.projection_name
order by start_epoch desc;
</code></pre>

2. 执行 select do_tm_task('moveout' , 'hanwenfeng.tb_rp_ct_86hl_forcast_source_day') 尝试把这个表在 WOS 中的数据强制写到 ROS 中，Vertica给出错误信息"ERROR:too many data partitions"
>找到问题根原了！原因在于: 这个表的分区数目超过了设计预期，导致 ROS Pushback 而无法拔 WOS 中的数据写入到 ROS 中，进而导致 LGE/AHM 差 CE 太多。
3. 于是解决问题就比较简单了: 提高这个表的分区粒度(比如从天变成月份)减少分区数， WOS 数据就可以正常写入到 ROS 中了。 

## 总结 ##
- 如果使用了实时分析模式，日常运维过程中需要重点监控AHM、LGE、CE等事务号的变化。
- 需要仔细设计表分区粒度。分区太多，可能会导致ROS Pushback(Vertica有参数控制每个projection在一个节点上 ROS Container 数量，缺省不能超过1024个)，但实时装载可能不能立即看到错误，因为它的数据先到内存，写磁盘是滞后的。
- 批量处理COPY和DML操作尽量加 direct 选项或 hint，避免用 WOS。



</body> </html>


