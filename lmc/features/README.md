# 特性索引

本目录存放按特性组织的文档：一个特性对应用户可见的一项能力，文档回答「这项能力由哪些
模块协作实现、数据如何流动、开关与限制是什么」。

模块内部的实现细节记在 [模块索引](../modules/README.md)，特性文档只写跨模块的协作与
用户可见行为，不复制模块细节。

## 命名与登记

- 文件名与用户在命令、配置项或文档中看到的特性名一致，例如 `self-heal.md`、
  `rebalance.md`、`thin-arbiter.md`；
- 新文档以 [特性文档模板](../meta/templates/feature-doc.md) 起步；
- 完成一篇文档后，在下表「已收录」中登记，并同步更新
  [知识库导航](../README.md#知识库地图)。

## 已收录

尚无特性级文档。

## 规划

下表是待撰写的特性清单。列入依据是「有独立的用户可见行为且跨多个模块」，顺序按「最常
被问到」到「较冷门」排列。

| 特性 | 主要涉及模块 | 文档应覆盖的重点 |
| --- | --- | --- |
| 副本与自愈 | `cluster/afr`、`heal`、`features/index` | 副本写入的事务边界、变更日志、自愈触发与观察方式 |
| 目录与文件分布 | `cluster/dht` | 文件定位算法、目录布局一致性、再平衡 |
| 纠删码 | `cluster/ec` | 分片布局、读写路径、修复，与副本方案的取舍 |
| 配额 | `features/quota`、`features/marker`、`quotad` | 统计口径、限额生效范围、超限时的行为 |
| 快照 | `features/snapview-client`、`features/snapview-server`、`snapd`、`glusterd` 的快照模块 | 快照卷的呈现方式、与源卷的关系、回滚 |
| 异地复制 | `geo-replication`、`features/changelog` | 同步链路、检查点、断点续传 |
| 变更追踪与文件清单 | `features/changelog`、`tools/glusterfind` | 变更日志的格式、全量与增量查询 |
| 位衰减检测 | `features/bit-rot`、`scrubd` | 签名存储位置、扫描与修复调度 |
| 数据缓存与失效 | `performance/md-cache`、`performance/io-cache`、`features/upcall` | 缓存生效条件、失效通知链路 |
| 大文件切分 | `features/shard` | 切分粒度、读聚合、与配额和自愈的相互影响 |
| 锁与租约 | `features/locks`、`features/leases` | 锁范围与语义、租约的回收 |
| 仲裁机制 | `features/arbiter`、`features/thin-arbiter` | 脑裂判定、仲裁节点的判定依据 |
| 命名空间隔离 | `features/namespace` | 命名空间视图的构造与限制 |
| 数据变换 | `features/cdc`（目录 `xlators/features/compress/`）、`features/sdfs` | 数据变换的落点与兼容性约束 |

## 相关文档

- [模块索引](../modules/README.md)
- [代码地图](../overview/code-map.md)
- [更新规则](../meta/update-rules.md)
