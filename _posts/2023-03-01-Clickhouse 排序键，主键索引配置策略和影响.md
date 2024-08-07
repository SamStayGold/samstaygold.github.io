---
title: Clickhouse排序键，主键索引配置策略及影响调研
date: 2023-03-01 19:01:03 +0800
categories: [大数据, Clickhouse]
tags: [clickhouse]
---

# Clickhouse排序键，主键索引配置策略及影响调研
## 说明
随着Clickhouse的使用率上升，CK相关的运维需求也越来越多，在这些需求中，表数据压缩率优化以及表查询效率优化两项，直接影响集群成本和业务体验，有必要充分探索。Clickhouse应用特殊的MergeTree表引擎，在绝大部分场景下，表大小和排序键强相关，查询效率和主键强相关，这里排序键和主键的配置规则，以及带来的影响，和传统数据库有很大的不同，因此写下本文，记录我对Clickhouse的主键配置策略的调研和测试结果。

## 参考文档
本文的大部分经验来自官方的主键索引优化文档（[Primary Indexes | ClickHouse Docs](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes)）

## 背景
目前clickhouse集群中有一张最大表`dws.t_dws_abtest_cube_di`表，这是一张从Hive摄入的表单，用于实验平台的业务查询，检查AB测试效果。

这张表留存了最近180天的数据，约830亿行，不算副本的话，总大小达到了2.5TB，而这张表在Hive中存储才不过300G，膨胀率达到了8.3倍，十分反常。
我们在检查字段压缩比时，发现user_id字段压缩比非常低。userid是一个32位哈希字符串，这种类型的数据，单条数据的压缩比较低，需要保证排序，尽量让多条一致的userid汇聚在一起，才能实现很好的压缩比。也就是说，排序键的配置，导致了user_id字段的数据膨胀。
业务侧建表时，排序键配置如下
`ORDER BY (test_id, test_tag, event, page_id, elem_id, area_id, day_gap, if_direct_pay, channel_id)`
Clickhouse在没有指定主键的情况下，会将排序键默认作为主键使用。业务方考虑了各场景下sql筛选字段的使用频率，做了排序，可以看到没有包含user_id作为排序键的一部分。

## 实验逻辑
那么，如何配置排序键和主键，才能在表压缩率和查询效率上求得平衡呢？
官方的最佳实践提到：
1. 主键必须是排序键的`Prefix`，也就是说，主键只能在尾部比排序键少几个字段。
2. 多主键/排序键场景下，在序列中排位越靠前的字段，筛选效果越明显，数据压缩比越高。
3. 平衡的场景下，应当按照基维来排序，基维越小的，排序应当越靠前。
4. 如有不常用的筛选字段，可以在排序键内配置，在主键内去掉，减少主键占用内存大小。

表单的基维如下：
| 字段名 | 维度量（Uniq数量） |
|--------|--------------------|
| test_id | 155 |
| test_tag | 352 |
| event | 3 |
| page_id | 5643 |
| elem_id | 459570 |
| area_id | 1047 |
| day_gap | 15 |
| if_direct_pay | 2 |
| channel_id | 2517 |
| user_id | 高基维 |

可以看到，`user_id`字段是基维最高的，clickhoues在`quota`内算不出来`count distinct`的结果，其次是`elem_id`，其他关键字段基围都相对较低。

我这边根据业务使用表的sql特点，表单字段特点，以及官方文档给的最佳实践，设计了几张实验表用于测试。
1. 不考虑查询效率，只追求压缩率，将user_id放到首位。对应表单dws.t_dws_abtest_cube_di_full_table_test，对应排序键配置`ORDER BY (user_id，test_id, test_tag, event, page_id, elem_id, area_id, day_gap, if_direct_pay, channel_id)`
2. 完全按照官方推荐，将现有的排序键重新排序，增加user_id作为排序键，放在末尾。对应表单dws.t_dws_abtest_cube_di_pk_test，对应排序键配置`ORDER BY (if_direct_pay, event, day_gap, test_id, test_tag, area_id, channel_id, page_id, elem_id, userid)`
3. 根据表单业务查询sql特点，去掉部分不需要的低基围排序键/主键，新增user_id作为排序键，但不作为主键，节约内存。对应表单dws.t_dws_abtest_cube_di_pk_test_1，对应排序键配置`ORDER BY (test_id, test_tag, area_id, channel_id, page_id, elem_id, userid)，对应主键配置PRIMARY KEY (test_id, test_tag, area_id, channel_id, page_id, elem_id)`
4. 根据基维对排序的影响，去掉部分不需要的低基围排序键/主键，去掉高基围的elem_id，提升user_id的压缩比，user_id不作为主键，节约内存。对应表单dws.t_dws_abtest_cube_di_pk_test_2，对应排序键配置`ORDER BY (test_id, test_tag, area_id, channel_id, page_id, userid)`，对应主键配置`PRIMARY KEY (test_id, test_tag, area_id, channel_id, page_id)`
5. 只保留业务查询的主要筛选字段，去掉其他不必要字段，提升user_id的压缩比，user_id不作为主键，节约内存。对应表单dws.t_dws_abtest_cube_di_pk_test_3，对应排序键配置`ORDER BY (test_id, test_tag, page_id, userid)`，对应主键配置`PRIMARY KEY (test_id, test_tag, page_id)`

为了减少不必要的数据存储，实验表均摄入最近1个月的数据，没有做1:1表。


## 实验过程及结果分析
### 数据量对比
| 表名 | 数据量（含副本） | 压缩比 |
|------|------------------|--------|
| t_dws_abtest_cube_di | 850GB | 11.70 |
| t_dws_abtest_cube_di_full_table_test_local | 206GB | 56.92 |
| t_dws_abtest_cube_di_pk_test_local | 829GB | 13.34 |
| t_dws_abtest_cube_di_pk_test_1_local | 838GB | 12.94 |
| t_dws_abtest_cube_di_pk_test_2_local | 656GB | 20.41 |
| t_dws_abtest_cube_di_pk_test_3_local | 500GB | 26.66 |

在数据摄入后，检查数据量及压缩比如上。可以看到，user_id在排序键序列中所在的位置，基本决定了表单数据的大小，以及压缩率的大小，越靠左，压缩比越高。如果user_id的排序处在一个基围比较高的字段（elem_id）背后，就算参与了排序，压缩比也无法实现很好的优化。
这里比较有意思的是if_direct_pay, event, day_gap三个低基围字段从排序键中去掉后，压缩效率反而有所下降，这也许是因为优先通过低基围字段排序后，user_id字段具有更高的重复率。
那么最优解就是userid开头作为排序/主键了么？别急，我们测试完查询的效率再看看。

### 查询效率
```sql
-- SQL1
select pt_day, test_tag
, count(distinct if(day_gap = 0 and if_xcx_pay = 1,user_id,null)) as pay_uv
, count(distinct user_id) as product_click_uv
from dws.t_dws_abtest_cube_di 
where test_tag in ( '8202302033001597', '8202302033001598') and test_id='7202302033000433' and pt_day>='2023-02-04' and pt_day<='2023-02-21' 
and page_id in ('pages/tabbar/home/index') 
group by test_tag, pt_day;

-- SQL2 基于sql1的实验，计算内容内包含element_id
select pt_day, test_tag
, count(distinct if(day_gap = 0 and if_xcx_pay = 1,user_id,null)) as pay_uv
, count(distinct if(elem_id like 'productCardNew%', user_id, null)) as product_click_uv
, count(distinct if(elem_id like 'channelNew%', user_id, null)) as channel_click_uv
from dws.t_dws_abtest_cube_di 
where test_tag in ( '8202302033001597', '8202302033001598') and test_id='7202302033000433' and pt_day>='2023-02-04' and pt_day<='2023-02-21' 
and page_id in ('pages/tabbar/home/index') 
group by test_tag, pt_day;

-- SQL3 基于sql1的实验，筛选内包含element_id
select pt_day, test_tag
, count(distinct if(day_gap = 0 and if_xcx_pay = 1,user_id,null)) as pay_uv
, count(distinct user_id) as product_click_uv
from dws.t_dws_abtest_cube_di 
where test_tag in ( '8202302033001597', '8202302033001598') and test_id='7202302033000433' and pt_day>='2023-02-04' and pt_day<='2023-02-21' 
and page_id in ('pages/tabbar/home/index') 
and elem_id like 'productCardNew%'
group by test_tag, pt_day;
```

| 表名 | sql1耗时 | sql1扫描行数 | sql2耗时 | sql2扫描行数 | sql3耗时 | sql3扫描行数 |
|------|----------|--------------|----------|--------------|----------|--------------|
| dws.t_dws_abtest_cube_di | 0.614 | 95.98 million | 0.662 | 96.00 million | 0.252 | 20.89 million |
| dws.t_dws_abtest_cube_di_full_table_test | 11.955 | 8.39 billion | 10.499 | 8.39 billion | 10.021 | 8.39 billion |
| dws.t_dws_abtest_cube_di_pk_test | 0.716 | 170.34 million | 0.846 | 170.34 million | 0.465 | 123.99 million |
| dws.t_dws_abtest_cube_di_pk_test_1 | 0.636 | 137.75 million | 0.735 | 137.75 million | 0.311 | 80.72 million |
| dws.t_dws_abtest_cube_di_pk_test_2 | 0.577 | 146.79 million | 0.661 | 146.79 million | 0.455 | 146.79 million |
| dws.t_dws_abtest_cube_di_pk_test_3 | 0.453 | 94.15 million | 0.579 | 94.15 million | 0.388 | 94.15 million |

可以看到，当业务常用的`test_id`，`test_tag`，`page_id`字段处在排序键/主键头部时，可以明显减少扫描行数，减少查询耗时。user_id排在开头的话，几乎需要扫描整表，效率极差。
如果筛选字段不在主键中，会影响一部分查询效率，但整体耗时业务还是能够接受的。
因为`test_id`，`test_tag`基本上会在所有业务sql中出现，`page_id`出现的概率也比较高（约占目前查询的30%），对于其他参与筛选的字段，可以不参与到排序中，实现存储效率和查询效率的平衡。而且，因为只需要从磁盘中加载更小量的数据，虽然扫描行数差不多，但执行耗时实际还是有优化的。

## 总结
综上，我们在设计clickhouse的local表结构时，需要充分考虑业务SQL类型，考虑字段特征（字段类型，基维，字段长度），并谨慎的设计表单的排序键和主键，求得表单整体压缩率与查询效率的平衡。
总的来说，排序键，主键索引配置可以遵循以下原则：
1. 业务最常用的筛选字段应当往前排。
2. 高基围，且字段数据本身重复率很低（比如哈希字符串）的，在不影响业务的情况下，尽量靠前排。
3. 如有多个字段配置联合排序键/主键，尽量按照基维从小到大的顺序排序。
4. 主键不需要的字段可以在配置中去除，减少索引的内存消耗。

这里CK最大表经过以上原则调整后，在不影响整体业务查询效率的情况下，可以提升2.2倍的压缩效率，最终表单大小将不到原来的1/2，收益还是比较可观的。
