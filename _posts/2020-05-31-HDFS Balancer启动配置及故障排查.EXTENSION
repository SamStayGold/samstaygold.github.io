---
title: HDFS Balancer启动配置及故障排查
date: 2020-05-31 21:15:03 +0800
categories: [大数据, HDFS]
tags: [hdfs]
---

之前启动集群Balancer减少热节点影响后，balancer进程运行约一小时就无法正常运行。产生`time out`错误，`threads quota is exceeded`错误以及`dfs.balancer.max-iteration-time` is passed错误。


在查询日志，进一步排查原因后，确认是balancer的参数配置不合理造成的。参照CSDN的[这篇文章](https://blog.csdn.net/qq_31598113/article/details/78196774)。对balancer参数做了一些修改。


Balancer启动命令如下：
```bash
hdfs balancer -D dfs.datanode.balance.max.concurrent.moves=24 -D dfs.balancer.moverThreads=10000 -D dfs.datanode.balance.bandwidthPerSec=30720000 -threshold 15
```


这里面参数配置参考如下：
`dfs.datanode.balance.max.concurrent.moves`=`numOfdisk` * 2

`moveThreads`=100-200 * `num Of DN`

`bandwidthPerSec` 按照实际的集群load情况参考配置，需要快速rebalance可以100M以上，在load高的时候可以调小一些运行。

最后给一个定时任务自动重启balancer的案例
```bash
#!/bin/bash
# Author：Sam Sheng
# 20200531

dates=`date +"%Y-%m-%d %H:%M:%S"`
echo "balance begin,time:$dates"

# Authenticate 鉴权，这里加了Kerberos才需要，如果没加入 kerberos，需要用 hdfs 用户执行脚本
echo xxxxxx | kinit hdfs

# 检查是否已经存在 balancer 进程了
hdfs dfs -test -e /system/balancer.id
if [ $? -eq 0 ];then
    previous_balancer_pid=$(ps aux | grep proc_balancer | grep -v grep | grep -v start_balancer|  awk '{print $2}')
    if [ -n "$previous_balancer_pid" ]
    then
        echo "Found pid for previous balancer, gonna kill $previous_balancer_pid"
        kill $previous_balancer_pid
    else
        echo "No existing balancer!"
    fi
fi

sleep 30s

# 配置Balancer带宽
hdfs dfsadmin -setBalancerBandwidth 268435456

# 启动 balancer进程
nohup hdfs balancer -Ddfs.datanode.balance.max.concurrent.moves=256 -Ddfs.balancer.max-size-to-move=32212254720  -Ddfs.balancer.dispatcherThreads=256  -Ddfs.balance.bandwidthPerSec=268435456 -threshold 3 -idleiterations 50 1>/root/samworkspace/balancer/fast_log/balancer-out.log 2>/root/samworkspace/balancer/fast_log/balancer-err.log &

sleep 30s

pid=$(ps aux | grep proc_balancer | grep -v grep | grep -v start_balancer|  awk '{print $2}')

# 如果没启动成功的话再试一次
if [ -z $pid ]
then
nohup hdfs balancer -Ddfs.datanode.balance.max.concurrent.moves=256 -Ddfs.balancer.max-size-to-move=32212254720  -Ddfs.balancer.dispatcherThreads=256  -Ddfs.balance.bandwidthPerSec=268435456 -threshold 5 -idleiterations 50 1>/root/samworkspace/balancer/fast_log/balancer-out.log 2>/root/samworkspace/balancer/fast_log/balancer-err.log &
fi

dates=`date +"%Y-%m-%d %H:%M:%S"`
echo "balance over,time:$dates"