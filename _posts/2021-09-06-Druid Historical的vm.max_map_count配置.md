---
title: Druid Historical的vm.max_map_count配置
date: 2021-09-06 00:00:00 +0800
categories: [大数据]
tags: [druid]
---

# 【配置】Druid Historical的vm.max_map_count配置


## 说明
Druid使用MMapfs来存储segments。对于操作系统，mmap的最大值限制在初始情况下往往不够druid的historical组件使用的。需要将相关配置提升至合理水平。

## 错误现象
如果max_map_count配置太小了，在启动historical，或者historical运行过程中，会报OOM错误，JVM进程直接停止的情况。而Druid historical的报错说明往往不甚明确，直接报OOM，排查过程会比较困难。

## 配置方式
临时配置可使用
```
sysctl -w vm.max_map_count=999999
```
为了避免重启机器后配置失效的情况，建议也使用永久的配置方式记录下
编辑`/etc/sysctl.conf`，添加或变更vm.max_map_count配置至相应数值

## 验证方式
执行临时配置命令后应该就生效了，验证变更是否生效可以通过以下命令处理
```
sysctl vm.max_map_count
```