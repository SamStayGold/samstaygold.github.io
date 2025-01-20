---
title: 为Hive3增加hive-cli能力，解锁MR引擎
date: 2023-01-10 00:00:00 +0800
categories: [大数据]
tags: [hive]
---

# 【Hive-Cli】为Hive3增加hive-cli能力，解锁MR引擎


Hive3默认使用beeline作为执行命令行客户端，当使用hive命令时，在拉起的hive.distro里，其实也是跑的beeline。
在Hive3中，默认禁用了beeline里的MR执行引擎，且beeline采用Hiveserver2的jdbc方式接入，与历史脚本的兼容性很差。

目前我司有部分SDK/DMP及mysql2hive流程依赖hive-cli及MR引擎的能力。

激活Hive3的hive-cli，并添加MR引擎的方法如下：

``` 
# 在dauser角色下，/data下创建time_functions_replace.sql文件，内容如下
CREATE TEMPORARY FUNCTION from_unixtime AS 'cn.xxxxx.hive.udf.UDFFromUnixTime' using jar 'hdfs://hdszcluster/user/hive/lib/hive-udfs-0.1.1.jar';

--CREATE TEMPORARY FUNCTION current_date AS 'cn.xxxxx.hive.udf.generic.GenericUDFCurrentDate' using jar 'hdfs://hdszcluster/user/hive/lib/hive-udfs-0.1.1.jar';

--CREATE TEMPORARY FUNCTION current_timestamp AS 'cn.xxxxx.hive.udf.generic.GenericUDFCurrentTimestamp' using jar 'hdfs://hdszcluster/user/hive/lib/hive-udfs-0.1.1.jar';

CREATE TEMPORARY FUNCTION unix_timestamp AS 'cn.xxxxx.hive.udf.generic.GenericUDFUnixTimeStamp' using jar 'hdfs://hdszcluster/user/hive/lib/hive-udfs-0.1.1.jar';

# 保证time_functions_replace.sql全员可读
chmod 644 /data/time_functions_replace.sql

# 打开hive.distro文件 需要root执行
vim /usr/hdp/3.1.0.0-78/hive/bin/hive.distro

# 注释掉这一行，去掉beeline取代hive-cli的逻辑
# USE_BEELINE_FOR_HIVE_CLI="true"

# 在 SERVICE_LIST="" 一行前添加下列代码，为hive-cli配置默认的MR环境
if [[ "$SERVICE" != beeline ]]; then
    HIVE_OPTS="$HIVE_OPTS --hiveconf hive.execution.engine=MR -i /data/time_functions_replace.sql "
fi
```