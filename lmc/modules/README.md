# 模块索引

本目录存放按模块组织的文档：一个模块对应代码中的一个 xlator 或一个子系统，文档回答
「这个模块内部如何组织、状态放在哪里、边界条件是什么」。

用户可见能力如何跨模块实现，记在 [特性索引](../features/README.md)，两类文档不重复
论述同一内容。

## 命名与登记

- 文件名与代码中的模块标识一致，例如 `afr.md`、`posix.md`、`glusterd.md`；
- 新文档以 [模块文档模板](../meta/templates/module-doc.md) 起步；
- 完成一篇文档后，在下表「已收录」中登记，并同步更新
  [知识库导航](../README.md#知识库地图)。

## 已收录

| 文档 | 覆盖内容 | 代码位置 |
| --- | --- | --- |
| [dict](dict.md) | 通用键值容器的结构、所有权规则、序列化格式与并发约束 | `libglusterfs/src/dict.c` |
| [inode 与 inode 表](inode.md) | inode 身份与路径解耦、dentry、ctx 槽位、引用与淘汰 | `libglusterfs/src/inode.c` |
| [fd](fd.md) | fd 的身份与生命周期、匿名 fd、fdtable 与整数 fd 分配 | `libglusterfs/src/fd.c` |

## 规划

下表是待撰写的模块清单，顺序按「被依赖程度」与「理解难度」排序，先写底座与数据通路，
再写上层功能。

| 模块 | 代码位置 | 文档应覆盖的重点 | 可复用的前置阅读 |
| --- | --- | --- | --- |
| `libglusterfs` 运行时设施 | `libglusterfs/src/event.c`、`syncop.c`、`statedump.c` | 事件循环、同步任务、状态转储 | [xlator 框架](../core/xlator-framework.md) |
| `storage/posix` | `xlators/storage/posix/src/` | handle 映射、扩展属性布局、落盘路径 | [术语表](../overview/glossary.md) |
| `protocol/client`、`protocol/server` | `xlators/protocol/{client,server}/src/` | 连接与握手、请求编解码、服务端解析与恢复 | [RPC 与传输](../core/rpc-and-transport.md) |
| `cluster/dht` | `xlators/cluster/dht/src/` | 文件到子卷的定位算法、目录布局、再平衡入口 | [架构总览](../overview/architecture.md) |
| `cluster/afr` | `xlators/cluster/afr/src/` | 事务框架、变更日志、自愈触发 | [特性索引](../features/README.md) |
| `cluster/ec` | `xlators/cluster/ec/src/` | 编码布局、读写路径、修复流程 | [特性索引](../features/README.md) |
| `mgmt/glusterd` | `xlators/mgmt/glusterd/src/` | 集群事务、store 持久化、volfile 生成、进程管理 | [volfile 与图构建](../core/volfile-and-graph.md) |
| `mount/fuse` | `xlators/mount/fuse/src/` | 内核接口映射、请求调度、挂载与卸载 | [调用栈与 FOP 模型](../core/call-stack-and-fop.md) |
| `api`（libgfapi） | `api/src/` | 不经过 FUSE 的客户端用法、句柄与并发模型 | [架构总览](../overview/architecture.md) |
| `cli` | `cli/src/` | 命令解析、RPC 调用、输出格式 | [架构总览](../overview/architecture.md#控制平面) |
| `performance/*` | `xlators/performance/` | 各缓存与合并类 translator 的生效条件与失效时机 | [xlator 框架](../core/xlator-framework.md) |
| `features/locks`、`features/leases` | `xlators/features/locks/src/` | 锁语义、锁所有者、与 NFS 的兼容 | [调用栈与 FOP 模型](../core/call-stack-and-fop.md) |
| `features/quota`、`features/marker` | `xlators/features/quota/src/`、`xlators/features/marker/src/` | 配额统计的存储格式与更新时机 | [特性索引](../features/README.md) |
| `features/changelog`、`features/index` | `xlators/features/changelog/src/`、`xlators/features/index/src/` | 变更日志格式与消费方式 | [特性索引](../features/README.md) |
| `features/bit-rot` | `xlators/features/bit-rot/src/` | 对象签名与扫描 | [特性索引](../features/README.md) |
| `features/shard` | `xlators/features/shard/src/` | 大文件切分与聚合读 | [特性索引](../features/README.md) |
| `geo-replication` | `geo-replication/` | 同步守护、检查点、与 changelog 的关系 | [特性索引](../features/README.md) |

## 相关文档

- [特性索引](../features/README.md)
- [代码地图](../overview/code-map.md)
- [更新规则](../meta/update-rules.md)
