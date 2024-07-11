---
title: 为HDP集群机器更改hostname
date: 2020-09-10 20:01:03 +0800
categories: [大数据, HDP]
tags: [hdp]
---

__参考文档__ [HDP DOC](https://docs.cloudera.com/HDPDocuments/Ambari-2.6.2.0/bk_ambari-administration/content/ch_changing_host_names.html)

在HDP集群中，机器的hostname往往以FQDN（Fully Qualified Domain Name）的形式存在。但是，如果需要修改机器的hostname，如何保证Amabri内的配置，以及集群各组件的配置正常呢？
这次测试环境的部署就遇到了需要从普通的hostname更改为FQDN的情况，这里记录一下。

以下是HDP集群机器更改hostname的步骤：
1. 准备修改hostname需要用到的json文件（注意，先不直接修改主机的hostname）
```
{
"cluster_name" : {
"old_hostname1" : "new_hostname1.dev.com",
"old_hostname2" : "new_hostname2.dev.com",
"old_hostname3" : "new_hostname3.dev.com"
}
}
```
2. 停止集群内正在运行的所有的任务以及任何等待的命令。
3. 停止集群内的所有组件。
4. 停止Ambari Agent以及Ambari Server
```
ambari-agent stop
ambari-server stop
```
5. 备份ambari数据库
6. 执行Ambari Server Hostname变更命令，确认所有的选项。
```
ambari-server update-host-names cluster_host.json
```
7. 执行成功后，可以开始修改各机器的的hostname了，可以直接在`/etc/hosts`内修改。
8. 若执行失败，建议可以看下json的语法是否正确，检查下新老hostname的规范。
9. 如果修改的是Amabri Server所在的机器hostname，需要在`/etc/ambari-agent/conf/ambari-agent.ini`下修改`hostname`字段。
10. 启动Ambari Agent以及Ambari Server
```
ambari-agent start
ambari-server start
```
11. 启动所有组件

PS：如果NameNode高可用开启的话需要在NameNode上执行：
```
hdfs zkfc -formatZK -force
```

PPS: 如果安装了Ranger，可能发生rangeradmin用户找不到导致ranger admin组件无法重启的情况。这是由于hosts变更后，mysql里rangeradmin用户的Host未升级导致的，登陆集群元数据库所在mysql，找到所有User的Host，若包含老的Hosts，替换为新的即可。