---
title: HDFS配置Disk Balancer平衡节点内的磁盘容量分布
date: 2020-08-31 21:20:03 +0800
categories: [大数据, HDFS]
tags: [hdfs]
---

## 背景
由于集群部分节点出现换盘等操作，部分磁盘的使用率没有提升起来。换盘一两周后还是维持在20%～30%使用率的水平，而其他热盘已经达到了85%，十分影响集群的正常运行。HDFS 3内引入了disk balancer功能，用于平衡节点内的磁盘容量分布，本SOP记录disk balancer相关操作，并以hd05，hd07，hd10节点为例，阐述disk balancer的功能。


## 命令与操作
### 使用report命令来找到适合运行disk balancer的节点
找到运行收益比较大的节点。这里输出top5：
```bash
[root@cvm-da-datasvr-hd09 diskBalancer]# hdfs diskbalancer -report -top 5
20/08/31 11:17:39 INFO command.Command: Processing report command
20/08/31 11:17:41 INFO balancer.KeyManager: Block token params received from NN: update interval=10hrs, 0sec, token lifetime=10hrs, 0sec
20/08/31 11:17:41 INFO block.BlockTokenSecretManager: Setting block keys
20/08/31 11:17:41 INFO balancer.KeyManager: Update block keys every 2hrs, 30mins, 0sec
20/08/31 11:17:41 INFO command.Command: Reporting top 5 DataNode(s) benefiting from running DiskBalancer.
Processing report command
Reporting top 5 DataNode(s) benefiting from running DiskBalancer.
1/5 hd07.prd.com[10.61.96.40:8010] - <be485c89-d1dc-4f89-bbb2-886bf3ea5e06>: 12 volumes with node data density 1.15.
2/5 hd05.prd.com[10.61.96.11:8010] - <b7382340-e3c3-40e2-b281-5e9a6537958c>: 12 volumes with node data density 0.99.
3/5 hd10.prd.com[10.61.96.38:8010] - <a6037189-12bc-49b7-b303-d69a948966b0>: 12 volumes with node data density 0.87.
4/5 hd11.prd.com[10.61.96.25:8010] - <6420a7c3-d017-497b-8c88-6afb75d6b46a>: 12 volumes with node data density 0.06.
5/5 hd04.prd.com[10.61.96.4:8010] - <98767b3a-14ae-4f9a-8594-21dd7719cb51>: 12 volumes with node data density 0.04.
```
从上面的输出流可以看到，hd07，hd05，hd10最适合运行disk balancer，潜在收益也最大。这三台节点最近刚好换过盘，和这里的输出应征上了。


### 使用plan命令来配置disk balancer计划：
`hdfs diskbalancer -plan hd05.prd.com`
用于配置某一节点的disk balancer计划，还可以添加以下参数：
| **COMMAND_OPTION** | **Description** |
|--------------------|------------------|
| `-out`             | Allows user to control the output location of the plan file. |
| `-bandwidth`       | Since datanode is operational and might be running other jobs, diskbalancer limits the amount of data moved per second. This parameter allows user to set the maximum bandwidth to be used. This is not required to be set since diskBalancer will use the default bandwidth if this is not specified. 默认10MB/sec |
| `-thresholdPercentage` | Since we operate against a snap-shot of datanode, the move operations have a tolerance percentage to declare success. If user specifies 10% and move operation is say 20GB in size, if we can move 18GB that operation is considered successful. This is to accommodate the changes in datanode in real time. This parameter is not needed and a default is used if not specified. 默认%10 |
| `-maxerror`        | Max error allows users to specify how many block copy operations must fail before we abort a move step. Once again, this is not a needed parameter and a system-default is used if not specified. 默认5 |
| `-v`               | Verbose mode, specifying this parameter forces the plan command to print out a summary of the plan on stdout. |
| `-fs`              | Specifies the namenode to use. if not specified default from config is used. |


 这里可以直接使用默认值：
```bash
[root@cvm-da-datasvr-hd09 diskBalancer]# hdfs diskbalancer -plan hd10.prd.com
20/08/31 11:04:13 INFO balancer.KeyManager: Block token params received from NN: update interval=10hrs, 0sec, token lifetime=10hrs, 0sec
20/08/31 11:04:13 INFO block.BlockTokenSecretManager: Setting block keys
20/08/31 11:04:13 INFO balancer.KeyManager: Update block keys every 2hrs, 30mins, 0sec
20/08/31 11:04:13 INFO planner.GreedyPlanner: Starting plan for Node : hd10.prd.com:8010
20/08/31 11:04:13 INFO planner.GreedyPlanner: Disk Volume set 0e2ae673-6250-49ab-a0d2-a783558520ce Type : DISK plan completed.
20/08/31 11:04:13 INFO planner.GreedyPlanner: Compute Plan for Node : hd10.prd.com:8010 took 33 ms
20/08/31 11:04:13 INFO command.Command: Writing plan to:
20/08/31 11:04:13 INFO command.Command: /system/diskbalancer/2020-Aug-31-11-04-13/hd10.prd.com.plan.json
Writing plan to:
/system/diskbalancer/2020-Aug-31-11-04-13/hd10.prd.com.plan.json
```


### 使用execute命令来执行计划
```
[root@cvm-da-datasvr-hd09 diskBalancer]# hdfs diskbalancer -execute /system/diskbalancer/2020-Aug-31-11-00-17/hd05.prd.com.plan.json
20/08/31 11:00:44 INFO command.Command: Executing "execute plan" command
```


### 使用query命令来查看计划运行状况
```bash
[root@cvm-da-datasvr-hd09 diskBalancer]# hdfs diskbalancer -query hd05.prd.com
20/08/31 11:01:14 INFO command.Command: Executing "query plan" command.
Plan File: /system/diskbalancer/2020-Aug-31-11-00-17/hd05.prd.com.plan.json
Plan ID: aa74d01299c254140b9a7b16f7880743d6dac75b
Result: PLAN_UNDER_PROGRESS
``` 


### 使用cancel命令来取消计划
```bash
[root@cvm-da-datasvr-hd09 diskBalancer]# hdfs diskbalancer -cancel /system/diskbalancer/2020-Aug-31-11-00-17/hd05.prd.com.plan.json
20/08/31 11:03:33 INFO command.Command: Executing "Cancel plan" command.
```

## 流程
使用report找到需要运行disk balancer的节点 -> 配置plan -> 执行plan -> 检查状况 -> 有问题cancel plan。

## P.S
disk balancer同集群balancer类似，也会对block的移动产生影响，建议在集群闲暇时段运行。

## 代码案例
最后这里分享一个DiskBalancer的拉起脚本，用法如下：
```bash
# 假设需要执行 Disk Balancer 的datanode 节点的 FQDN 是 hddata001.prd.com
# 生成 Balance Plan
sh diskbalancer.sh plan hddata001.prd.com
# 生成完毕后可以执行 Plan
sh diskbalancer.sh execute hddata001.prd.com
# 执行过程中可以检查下进度
sh diskbalancer.sh check hddata001.prd.com
# 或者可以取消plan
sh diskbalancer.sh cancel hddata001.prd.com
```

```bash
#!/bin/bash
# Filename: diskbalancer.sh
# Author： Sam Sheng

# Kerberos鉴权，如果不是kerberos集群，则需要用 hdfs 用户执行
export KRB5CCNAME=/tmp/ktb5cc_hdfs
echo xxxxxx | kinit hdfs
if [ "$1" = "plan" ]; then
    echo "Planning..."
    plan_dir=$(hdfs diskbalancer -plan $2 -thresholdPercentage 2 -bandwidth 50 2>&1 | tail -n 1)
    echo "Diskbalancer planned for $2 at: ${plan_dir}"
    # 输出执行计划路径到本地文件用于后续跟进
    echo "${plan_dir}" > plan_dir.txt
fi

if [ "$1" = "execute" ]; then
    echo "Executing..."
    hdfs diskbalancer -execute $(cat plan_dir.txt)
fi

if [ "$1" = "check" ]; then
    echo "Checking..."
    hdfs diskbalancer -query $2 -v
fi

if [ "$1" = "cancel" ]; then
    echo "Cancelling..."

    hdfs diskbalancer -cancel $(cat plan_dir.txt)
fi
```