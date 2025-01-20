---
title: 处理DB configs consistency check报错的问题
date: 2021-03-15 00:00:00 +0800
categories: [大数据]
tags: [amabri修复]
---

# 【amabri修复】处理DB configs consistency check报错的问题


有时卸载组件后ambari不能正常处理组件的配置，导致配置项留存在ambari元数据库中成为孤儿配置。并在变更时弹窗警告，影响变更进程。

检查/var/log/ambari-server/ambari-server-check-database.log的日志，可以确认哪些配置项没有map上。确认后，可以在ambari元数据库中执行以下命令处理数据库问题。

```sql
CREATE  TABLE orphaned_configs (
config_id` bigint(20) NOT NULL,
type_name` varchar(100) NOT NULL,
 `version_tag` varchar(100) NOT NULL,
PRIMARY KEY (`config_id`));

insert into orphaned_configs 
(select config_id,type_name,version_tag from clusterconfig where unmapped != 1 and type_name not in ('cluster-env') and config_id not in (
SELECT
cc.config_id 
FROM clusterconfig cc, serviceconfig sc, serviceconfigmapping scm
WHERE cc.type_name != 'cluster-env'
AND cc.config_id = scm.config_id
AND scm.service_config_id = sc.service_config_id));

Update clusterconfig a INNER JOIN orphaned_configs b  on a.config_id=b.config_id set a.unmapped = 1 ;
```