---
title: Coordinator的Balancer机制与在线更新配置
date: 2022-02-28 00:00:00 +0800
categories: [大数据]
tags: [druid]
---

# 【Druid】Coordinator的Balancer机制与在线更新配置




---
星期中间的时候多次接到我司Druid Historical hddruidcold002节点的告警，磁盘使用率高。我司当前Druid的数据量的确很大，但数据是分散在3台冷节点，4台热节点上的，且冷节点磁盘容量都是TB级别的，怎么会出现单个druid historical磁盘使用率高的问题呢？

仔细探究了下Druid中节点数据量平衡相关的配置和原理：
Druid中，由Coordinator作为最小存储单元（即Segments）的调度器，负责维护存储的元数据，负责分配每个Segment的存放位置。每隔60s（由```druid.coordinator.period```控制），Coordinaor会执行一遍自身的主流程，检测一遍当前的存储分配结构，结合每个Datasource的存储策略，以及节点的数据量，来判断当前存储的结构是否符合需求，有哪些segments需要做调整。之后，便会交由```io.druid.server.coordinator.helper.DruidCoordinatorBalancer```来处理平衡逻辑，移动需要的segments，由```io.druid.server.coordinator.CuratorLoadQueuePeon```处理segments，完成重装载。

在节点平衡中，平衡类型和平衡速率是非常重要的参数，平衡类型不当，会造成节点失衡，平衡速率太慢，每次Balancer移动的块数量太少，也会造成节点在长期的摄入，数据源存储策略调整的流程中逐渐失衡。

调节节点平衡策略可以通过：```druid.coordinator.balancer.strategy``` 配置调整
目前支持cost，diskNormalized，random三种，cost策略会同时考虑查询效能，缓存分配的情况，但是可能会造成不平均的情况。random只是随机的分配平衡的节点。而diskNormalized则会尽量保证节点的磁盘容量消耗趋于平衡。

调节平衡速率可以通过：```druid.coordinator.maxSegmentsToMove``` 来调节
这个配置控制每次可以移动的Segments数量。

其中，maxSegmentsToMove是可以在线调整的，可以选择在集群闲暇时增加maxSegmentsToMove的值，最大化平衡效率，而在集群查询任务紧张时，减少这个值，减少平衡带来的性能影响。
在线调整可以通过：
`http://<COORDINATOR_IP>:<PORT>/druid/coordinator/v1/config`
接口完成
支持的参数如下：
|Property |Description |Default |
|:--|
|`millisToWaitBeforeDeleting` |How long does the coordinator need to be active before it can start removing (marking unused) segments in metadata storage. |900000 (15 mins) |
|`mergeBytesLimit` |The maximum total uncompressed size in bytes of segments to merge. |524288000L |
|`mergeSegmentsLimit` |The maximum number of segments that can be in a single [append task](https://druid.io/docs/0.12.1/ingestion/tasks.html). |100 |
|`maxSegmentsToMove` |The maximum number of segments that can be moved at any given time. |5 |
|`replicantLifetime` |The maximum number of coordinator runs for a segment to be replicated before we start alerting. |15 |
|`replicationThrottleLimit` |The maximum number of segments that can be replicated at one time. |10 |
|`emitBalancingStats` |Boolean flag for whether or not we should emit balancing stats. This is an expensive operation. |false |
|`killDataSourceWhitelist` |List of dataSources for which kill tasks are sent if property  `druid.coordinator.kill.on`  is true. This can be a list of comma-separated dataSources or a JSON array. |none |
|`killAllDataSources` |Send kill tasks for ALL dataSources if property  `druid.coordinator.kill.on`  is true. If this is set to true then  `killDataSourceWhitelist`  must not be specified or be empty list. |false |
|`killPendingSegmentsSkipList` |List of dataSources for which pendingSegments are NOT cleaned up if property  `druid.coordinator.kill.pendingSegments.on`  is true. This can be a list of comma-separated dataSources or a JSON array. |none |
|`maxSegmentsInNodeLoadingQueue` |The maximum number of segments that could be queued for loading to any given server. This parameter could be used to speed up segments loading process, especially if there are "slow" nodes in the cluster (with low loading speed) or if too much segments scheduled to be replicated to some particular node (faster loading could be preferred to better segments distribution). Desired value depends on segments loading speed, acceptable replication time and number of nodes. Value 1000 could be a start point for a rather big cluster. Default value is 0 (loading queue is unbounded) |0 |

将需要调整的参数，放在json内提交即可：
```json
{
  "millisToWaitBeforeDeleting": 900000,
  "mergeBytesLimit": 100000000,
  "mergeSegmentsLimit" : 1000,
  "maxSegmentsToMove": 5,
  "replicantLifetime": 15,
  "replicationThrottleLimit": 10,
  "emitBalancingStats": false,
  "killDataSourceWhitelist": ["wikipedia", "testDatasource"]
}
```

在生产环境中，还需要使用kinit鉴权，并且在curl中加入negotiate参数，因此，为了简化操作，我们在hdcommon002节点的/root/DruidCoordinatorDynamicConfigurator下，新增了configurator.sh脚本，变更json内的配置项和内容，执行脚本就可以方便的在线变更参数了！