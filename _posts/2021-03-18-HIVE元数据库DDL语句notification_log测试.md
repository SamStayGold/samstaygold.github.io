---
title: HIVE元数据库DDL语句notification_log测试
date: 2021-03-18 00:00:00 +0800
categories: [大数据]
tags: [hive]
---

# HIVE元数据库DDL语句notification_log测试




---
# DDL语句notification_log测试

## notification_log表说明
开启了`hive.metastore.event.listeners`的`org.apache.hive.hcatalog.listener.DbNotificationListener`配置后，对于元数据的操作，比如CREATE，ALTER和DROP会在notification_log表中留下事件记录，本来该配置用于Hive自带的replication特性中。本文意在探索如何解析，并利用这个功能，让我们能够感知到用户对表的变更操作，并在集群迁移过程中能够及时反应。
hive元数据库，notification_log表结构如下
```
MySQL [hive]> desc notification_log;
 ---------------- -------------- ------ ----- --------- ------- 
| Field          | Type         | Null | Key | Default | Extra |
 ---------------- -------------- ------ ----- --------- ------- 
| NL_ID          | bigint(20)   | NO   | PRI | NULL    |       |
| EVENT_ID       | bigint(20)   | NO   | MUL | NULL    |       |
| EVENT_TIME     | int(11)      | NO   |     | NULL    |       |
| EVENT_TYPE     | varchar(32)  | NO   |     | NULL    |       |
| CAT_NAME       | varchar(256) | YES  |     | NULL    |       |
| DB_NAME        | varchar(128) | YES  |     | NULL    |       |
| TBL_NAME       | varchar(256) | YES  |     | NULL    |       |
| MESSAGE        | longtext     | YES  |     | NULL    |       |
| MESSAGE_FORMAT | varchar(16)  | YES  |     | NULL    |       |
 ---------------- -------------- ------ ----- --------- ------- 
9 rows in set (0.00 sec)
```
可以看到，表结构由事件信息，DB，TABLE信息及消息体组成。接下来，我们将通过建表，Insert分区，修改表结构，DROP分区等一系列操作，了解notification_log表的消息结构。

## 建表操作
在tmp_db中创建临时测试表
```sql
CREATE TABLE `notification_test`(
`db` string COMMENT 'hive db name',
`tbl` string COMMENT 'hive table name',
`first_partition` string COMMENT 'name of the first partition',
`c_time` string COMMENT 'created time')
PARTITIONED BY (`day` string)
```
在元数据库中搜索刚才创建临时表的event
```
MySQL [hive]> select * from notification_log where DB_NAME='tmp_db' and TBL_NAME='notification_test';
```
我们关注EVENT_TIME，EVENT_TYPE，CAT_NAME，MESSAGE字段
EVENT_TIME:1616068620
EVENT_TIME记录了事件发生的时间，是一个UNIX TIMESTAMP，解析出来是2021年3月18日星期四晚上7点57分 GMT 08:00，正是我们创建表的时间。
EVENT_TYPE:CREATE_TABLE
这个就很清晰了，EVENT_TYPE记录了是什么类型的操作。
CAT_NAME:hive
这里推断是执行用户，一会可以试试看。
```
MESSAGE：
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",4,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",2,{\"transient_lastDdlTime\":\"1616068620\",\"bucketing_version\":\"2\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"13\":{\"rec\":{\"1\":{\"map\":[\"str\",\"lst\",0,{}]}}},\"14\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1}}","timestamp":1616068620,"files":[]}
```
可以看出，MESSAGE是一个json-0.2规格的raw字符串，其中包涵了我们需要的各类信息，包括操作时间，操作库表，操作表ObjJSON等。

## 确认INSERT OVERWRITE新建分区这样的操作有没有记录
```
INSERT OVERWRITE TABLE notification_test SELECT * from diuser.meta_hive_info WHERE day='2021-03-18';
INSERT OVERWRITE TABLE notification_test SELECT * from diuser.meta_hive_info WHERE day='2021-03-17';
```
果然，EVENT_TYPE:ADD_PARTITION，新增分区有记录的
```
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",4,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",2,{\"transient_lastDdlTime\":\"1616068620\",\"bucketing_version\":\"2\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","timestamp":1616070330,"partitions":[{"day":"2021-03-18"}],"partitionListJson":["{\"1\":{\"lst\":[\"str\",1,\"2021-03-18\"]},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"notification_test\"},\"4\":{\"i32\":1616070330},\"5\":{\"i32\":0},\"6\":{\"rec\":{\"1\":{\"lst\":[\"rec\",4,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"7\":{\"map\":[\"str\",\"str\",3,{\"transient_lastDdlTime\":\"1616070330\",\"totalSize\":\"2941751\",\"numFiles\":\"5\"}]},\"9\":{\"str\":\"hive\"}}"],"partitionFiles":[{"partitionName":"day=2021-03-18","files":["hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18/000000_0###","hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18/000001_0###","hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18/000002_0###","hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18/000003_0###","hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18/000004_0###"]}]}
```

## 确认Alter Table操作的记录
Alter Table有两种，一种是Altertable变更表结构，一种是Alter table drop分区，两周都试一下

```
ALTER TABLE notification_test ADD COLUMNS (u_time string);
```
检查notification_log表，出现了ALTER_TABLE操作。
```
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjBeforeJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",4,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",2,{\"transient_lastDdlTime\":\"1616068620\",\"bucketing_version\":\"2\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","tableObjAfterJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",5,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}},{\"1\":{\"str\":\"u_time\"},\"2\":{\"str\":\"string\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",4,{\"transient_lastDdlTime\":\"1616070641\",\"bucketing_version\":\"2\",\"last_modified_by\":\"hive\",\"last_modified_time\":\"1616070641\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","isTruncateOp":"false","timestamp":1616070641,"writeId":0}
```
可以看到，MESSAGE的json体中，携带有tableObjBeforeJson，tableObjAfterJson，记录了前后的库表元数据变化。

```
ALTER TABLE notification_test DROP IF EXISTS PARTITION (day=‘2021-03-17’);
```
对于drop partition，有特殊的EVENT_TYPE，DROP_PARTITION，其MESSAGE内容如下
```
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",5,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}},{\"1\":{\"str\":\"u_time\"},\"2\":{\"str\":\"string\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",4,{\"last_modified_time\":\"1616070641\",\"transient_lastDdlTime\":\"1616070641\",\"bucketing_version\":\"2\",\"last_modified_by\":\"hive\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","timestamp":1616071104,"partitions":[{"day":"2021-03-17"}]}
```
有时，开发还会对单partition的元数据进行变更操作，让我们了解下有什么不同：
```
ALTER TABLE notification_test PARTITION (day='2021-03-17') ADD COLUMNS (new_timestamp string);
```
对于单partition/多partitions的变更操作，可以通过EVENT_TYPE为ALTER_PARTITION过滤。
MESSAGE如下
```
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",5,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}},{\"1\":{\"str\":\"u_time\"},\"2\":{\"str\":\"string\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",4,{\"last_modified_time\":\"1616070641\",\"transient_lastDdlTime\":\"1616070641\",\"bucketing_version\":\"2\",\"last_modified_by\":\"hive\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","isTruncateOp":"false","timestamp":1616071558,"writeId":0,"keyValues":{"day":"2021-03-18"},"partitionObjBeforeJson":"{\"1\":{\"lst\":[\"str\",1,\"2021-03-18\"]},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"notification_test\"},\"4\":{\"i32\":1616070330},\"5\":{\"i32\":0},\"6\":{\"rec\":{\"1\":{\"lst\":[\"rec\",4,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"7\":{\"map\":[\"str\",\"str\",3,{\"transient_lastDdlTime\":\"1616070330\",\"totalSize\":\"2941751\",\"numFiles\":\"5\"}]},\"9\":{\"str\":\"hive\"},\"10\":{\"i64\":0}}","partitionObjAfterJson":"{\"1\":{\"lst\":[\"str\",1,\"2021-03-18\"]},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"notification_test\"},\"4\":{\"i32\":1616070330},\"5\":{\"i32\":0},\"6\":{\"rec\":{\"1\":{\"lst\":[\"rec\",5,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}},{\"1\":{\"str\":\"new_timestamp\"},\"2\":{\"str\":\"string\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test/day=2021-03-18\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"7\":{\"map\":[\"str\",\"str\",5,{\"transient_lastDdlTime\":\"1616071558\",\"totalSize\":\"2941751\",\"last_modified_by\":\"hive\",\"last_modified_time\":\"1616071558\",\"numFiles\":\"5\"}]},\"9\":{\"str\":\"hive\"},\"10\":{\"i64\":0}}"}
```

## DROP TABLE
```
DROP TABLE IF EXISTS notification_test;
```
检查notification_log表，出现DROP_TABLE操作，MESSAGE如下
```
{"server":"thrift://hdcom01.prd.com:9083,thrift://hdcom02.prd.com:9083,thrift://hdcom03.prd.com:9083","servicePrincipal":"hive/_HOST@xxxxx.CN","db":"tmp_db","table":"notification_test","tableType":"MANAGED_TABLE","tableObjJson":"{\"1\":{\"str\":\"notification_test\"},\"2\":{\"str\":\"tmp_db\"},\"3\":{\"str\":\"hive\"},\"4\":{\"i32\":1616068620},\"5\":{\"i32\":0},\"6\":{\"i32\":0},\"7\":{\"rec\":{\"1\":{\"lst\":[\"rec\",5,{\"1\":{\"str\":\"db\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive db name\"}},{\"1\":{\"str\":\"tbl\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"hive table name\"}},{\"1\":{\"str\":\"first_partition\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"name of the first partition\"}},{\"1\":{\"str\":\"c_time\"},\"2\":{\"str\":\"string\"},\"3\":{\"str\":\"created time\"}},{\"1\":{\"str\":\"u_time\"},\"2\":{\"str\":\"string\"}}]},\"2\":{\"str\":\"hdfs://hdcluster/warehouse/tablespace/managed/hive/tmp_db.db/notification_test\"},\"3\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcInputFormat\"},\"4\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcOutputFormat\"},\"5\":{\"tf\":0},\"6\":{\"i32\":-1},\"7\":{\"rec\":{\"2\":{\"str\":\"org.apache.hadoop.hive.ql.io.orc.OrcSerde\"},\"3\":{\"map\":[\"str\",\"str\",1,{\"serialization.format\":\"1\"}]}}},\"8\":{\"lst\":[\"str\",0]},\"9\":{\"lst\":[\"rec\",0]},\"10\":{\"map\":[\"str\",\"str\",0,{}]},\"11\":{\"rec\":{\"1\":{\"lst\":[\"str\",0]},\"2\":{\"lst\":[\"lst\",0]},\"3\":{\"map\":[\"lst\",\"str\",0,{}]}}},\"12\":{\"tf\":0}}},\"8\":{\"lst\":[\"rec\",1,{\"1\":{\"str\":\"day\"},\"2\":{\"str\":\"string\"}}]},\"9\":{\"map\":[\"str\",\"str\",4,{\"last_modified_time\":\"1616070641\",\"transient_lastDdlTime\":\"1616070641\",\"bucketing_version\":\"2\",\"last_modified_by\":\"hive\"}]},\"12\":{\"str\":\"MANAGED_TABLE\"},\"15\":{\"tf\":0},\"17\":{\"str\":\"hive\"},\"18\":{\"i32\":1},\"19\":{\"i64\":0}}","timestamp":1616071666}
```

## 结论
我们可以通过扫描EVENT_TYPE的方式，感知到集群内的ALTER_TABLE，ALTER_PARTITION操作。
```
MySQL [hive]> select EVENT_TYPE, count(1) from (select * from notification_log where EVENT_TYPE in ("ALTER_PARTITION","ALTER_TABLE") and  DB_NAME not in ("tmp_db", "test_db") and EVENT_TIME > 1615996800) a group by EVENT_TYPE;
 ----------------- ---------- 
| EVENT_TYPE      | count(1) |
 ----------------- ---------- 
| ALTER_PARTITION |    21714 |
| ALTER_TABLE     |      280 |
 ----------------- ---------- 
2 rows in set (0.05 sec)
```
可以看到仅一日就有多次变更操作，变更次数最多的表如下
```
MySQL [hive]> select EVENT_TYPE, DB_NAME, TBL_NAME, count(1) as CHANGE_TIME from (select * from notification_log where EVENT_TYPE in ("ALTER_PARTITION","ALTER_TABLE") and  DB_NAME not in ("tmp_db", "test_db") and EVENT_TIME > 1615996800) a group by EVENT_TYPE, DB_NAME, TBL_NAME order by CHANGE_TIME;

| ALTER_PARTITION | diuser               | t_policy_deduct                                         |         398 |
| ALTER_PARTITION | dwd                  | dwd_guanggao_policy_new_id_di                           |         444 |
| ALTER_PARTITION | dwd                  | dwd_guanggao_policy_new_wtag_di                         |         444 |
| ALTER_PARTITION | dws                  | t_dws_policy_push_retain_cube_di                        |         460 |
| ALTER_PARTITION | bi_analys            | adm_daizhifu_uv_di                                      |         579 |
| ALTER_PARTITION | adm                  | adm_search_pay_di                                       |         877 |
| ALTER_PARTITION | adm                  | adm_wedrive_deal_di                                     |        1568 |
| ALTER_PARTITION | adm                  | adm_wedrive_boss_dail_mail_di                           |        1568 |
| ALTER_PARTITION | adm                  | adm_health_unionuser_cross_rpt_di                       |        1613 |
| ALTER_PARTITION | dwd                  | dwd_agent_customer_lite_dd                              |        1655 |
| ALTER_PARTITION | adm                  | adm_health_modx_info_di                                 |        4167 |
 ----------------- ---------------------- --------------------------------------------------------- ------------- 
```
应该是程序变更的，需要进一步细查过滤。