# inode 与 inode 表

`inode_t` 是 GlusterFS 进程内对「一个文件对象」的表示。它把不稳定的用户路径与稳定的
GFID 解耦：路径只用于解析，解析结果按 GFID 缓存在 inode 表里，同一文件经不同路径反复
访问时命中同一条 inode 记录。inode 表同时保存目录项（dentry）关系，使「由路径反查文件」
与「由文件反查路径」都能在内存中完成。

## 目录

- 概览
- 核心概念
- 数据结构
- 工作流程
- 关键分支与边界条件
- 并发与锁
- 配置项
- 与其他模块的交互
- 可观测性
- 代码位置

## 概览

inode 层承担三件事：

| 职责 | 说明 |
| --- | --- |
| 身份归一 | 同一 GFID 在表内只有一条记录，路径变化不影响身份 |
| 引用管理 | 用引用计数与 nlookup 计数决定记录何时缓存、何时淘汰 |
| 私有数据挂载点 | 每个 xlator 可以在 inode 上挂自己的上下文，互不干扰 |

inode 表不是全局单例。它按使用场景分别创建：FUSE 挂载进程为每个图建一张，服务端为每个
已连接客户端建一张，`libgfapi` 与部分 translator 也会自行创建，见「与其他模块的交互」。

## 核心概念

**GFID。** 文件在卷内的全局唯一标识，见 [术语表](../overview/glossary.md#命名与标识)。
inode 的哈希键就是 GFID。

**dentry（目录项）。** 一条「父目录 inode + 名字 → inode」的映射。一个文件可以有多条
dentry（硬链接、以及同一文件被多个路径访问），它们挂在 inode 的 `dentry_list` 上。

**nlookup 与 ref 的区别。** 这是最容易混淆的一对计数：

| 计数 | 含义 | 归零时 |
| --- | --- | --- |
| `nlookup` | 上层（通常是内核或调用方）还持有多少个「查表所得」的引用 | 表示路径解析缓存可以丢弃 |
| `ref` | 进程内的内存引用计数 | 表示可以进入淘汰流程 |

回收时先看 `ref`：`ref` 减到 0 后，若 `nlookup` 仍大于 0，记录被放进 LRU 继续缓存；若
`nlookup` 也为 0，则进入待销毁队列（`__inode_unref`，`libglusterfs/src/inode.c:457`）。

**inode ctx。** 每个 xlator 可以在 inode 上缓存自己的私有数据，通过
`inode_ctx_set0`/`inode_ctx_get0`（第一槽）与 `inode_ctx_set1`/`inode_ctx_get1`（第二槽）
读写。槽位不是动态字典，而是一个按 xlator 在图中位置预先算好的定长数组
（见「数据结构」）。

## 数据结构

### inode 表

`inode_table_t`（`libglusterfs/src/glusterfs/inode.h:35`）的核心字段：

| 字段 | 含义 |
| --- | --- |
| `inode_hash` / `inode_hashsize` | GFID → inode 的哈希桶 |
| `name_hash` / `dentry_hashsize` | (父目录, 名字) → dentry 的哈希桶 |
| `active` / `active_size` | 正在被 FOP 使用的 inode 链表 |
| `lru` / `lru_size` | 仍在缓存、可被回收的 inode 链表，`lru.next` 是最新的 |
| `purge` / `purge_size` | 已解除哈希、等待销毁的 inode 链表 |
| `invalidate` / `invalidate_size` | 等待通知上层失效的 inode 队列 |
| `lru_limit` | LRU 上限，为 0 表示不按容量回收 |
| `xl`、`root`、`root_id`、`root_level` | 所属 xlator、根 inode 及其在 graph 中的位置 |
| `ctxcount` | inode ctx 数组的槽位数量 |
| `invalidator_fn`、`invalidator_xl` | 需要通知外部失效时调用的回调与其所属 xlator |

### inode

`inode_t`（`libglusterfs/src/glusterfs/inode.h:105`）的核心字段：

| 字段 | 含义 |
| --- | --- |
| `table` | 所属 inode 表 |
| `gfid` | 唯一标识 |
| `nlookup`、`kids` | 查表引用数、子目录项数（均为原子计数） |
| `ref` | 内存引用计数 |
| `fd_count`、`active_fd_count` | 打开的 fd 数、仍活跃的 fd 数 |
| `fd_list`、`dentry_list` | 该 inode 上的 fd 链表与 dentry 链表 |
| `hash`、`list` | 用于挂入哈希桶与 active/lru/purge 链表 |
| `ia_type` | 文件类型，决定 `release` 还是 `releasedir` 回调 |
| `ns_inode` | 指向所属命名空间的 inode |
| `_ctx[]` | 柔性数组，每个 xlator 一个槽位 |

### dentry 与 ctx 槽位

`dentry_t`（`libglusterfs/src/glusterfs/inode.h:69`）由 `inode_list`、`hash`、`inode`、
`parent` 与柔性数组 `name[]` 组成。`struct _inode_ctx`
（`libglusterfs/src/glusterfs/inode.h:78`）每个槽位保存两组「64 位值或指针」，第一组的键
位置记录归属的 xlator；还有一个仅在调试时有意义的 `ref` 计数，用于在 statedump 中定位
引用泄漏的 xlator。

槽位数量在表创建时确定：`ctxcount = xl->level + xl->child_count + 1`
（`libglusterfs/src/inode.c:1698`），即图中 xlator 的总数。槽位下标由
`inode_get_ctx_index`（`libglusterfs/src/inode.c:144`）按 xlator 的 `level` 与
`xl_id` 换算，因此 ctx 读写是常数时间。

## 工作流程

### 建立表

`inode_table_new`（`libglusterfs/src/inode.c:1794`）转调
`inode_table_with_invalidator`（`libglusterfs/src/inode.c:1671`），后者完成三件事：

| 事项 | 规则 | 位置 |
| --- | --- | --- |
| 槽位数量 | `level + child_count + 1` | `libglusterfs/src/inode.c:1698` |
| inode 哈希桶 | 必须是 2 的幂，小于 65536 时抬到 65536 | `libglusterfs/src/inode.c:1714` |
| dentry 哈希桶 | 传入 0 时取素数 14057 | `libglusterfs/src/inode.c:1700` |

根 inode 在表创建时一并生成，GFID 为全零值，且始终留在 active 链表中：对根 inode 的
`unref` 是空操作（`libglusterfs/src/inode.c:457`）。

### 查找

| 场景 | 接口 | 说明 |
| --- | --- | --- |
| 已知 GFID | `inode_find`（`libglusterfs/src/inode.c:924`） | 走 `inode_hash`，命中后取引用 |
| 已知父目录与名字 | `inode_grep`（`libglusterfs/src/inode.c:759`） | 走 `name_hash` 找 dentry |
| 已知路径 | `inode_from_path`、`inode_resolve` | 逐段解析后复用上面的接口 |
| 由 inode 反查路径 | `inode_path`（`libglusterfs/src/inode.c:1521`） | 沿 dentry 逐级上溯拼接 |

### 关联

`inode_link`（`libglusterfs/src/inode.c:1069`）是 lookup 结果落地的关键接口，语义由
`__inode_link`（`libglusterfs/src/inode.c:953`）实现：

1. 校验父目录必须是目录、必须与本 inode 属于同一张表、名字非空且不含 `/`；
2. inode 尚未入哈希时用 `iatt->ia_gfid` 计算桶位；若该 GFID 已在表中存在，则复用已有
   inode 而不是新建；
3. GFID 为全零时映射到根 inode；
4. 若该 (父目录, 名字) 尚无 dentry 或指向别的 inode，则新建 dentry 并做环路检查；
5. 关联成功后对结果取一次引用。

### 引用与回收

状态迁移由三个内部函数完成，它们在 `ref` 减到 0 时按 `nlookup` 分流
（`libglusterfs/src/inode.c:402`、`:409`、`:427`）：

| 函数 | 迁移 | 触发条件 |
| --- | --- | --- |
| `__inode_activate` | 移入 active 链表 | 开始被 FOP 使用 |
| `__inode_passivate` | 移入 LRU 链表 | `ref` 归零但 `nlookup > 0`，保留缓存 |
| `__inode_retire` | 移入 purge 链表并解除哈希 | `ref` 与 `nlookup` 均为 0，等待销毁 |

`inode_table_prune`（`libglusterfs/src/inode.c:1555`）在每次引用变化与关联之后被调用，
负责两件事：按 `lru_limit` 从 LRU 链尾回收（`lru.next` 是最近使用项），以及把 purge
链表整体摘出后销毁。若表配置了 `invalidator_fn` 且被选中的 inode 仍被上层持有
（`nlookup > 0`），则改为回调通知上层失效，而不是直接销毁。

### 失效

`inode_invalidate`（`libglusterfs/src/inode.c:1194`）用于 xlator 判定自己缓存的 inode
数据已过期、需要重建的场景。配置了 `invalidator_fn` 的表（典型是 FUSE）会借此通知内核
作废对应缓存条目。

## 关键分支与边界条件

| 条件 | 行为 | 位置 |
| --- | --- | --- |
| 跨 inode 表链接 | 断言失败，避免难以排查的状态错乱 | `libglusterfs/src/inode.c:953` |
| 父 inode 不是目录 | 断言失败并返回 NULL | `libglusterfs/src/inode.c:953` |
| 名字为空或含 `/` | 拒绝关联 | `libglusterfs/src/inode.c:1069` |
| `iatt->ia_gfid` 为全零 | 拒绝入哈希 | `libglusterfs/src/inode.c:953` |
| dentry 形成环路 | 删除新建的 dentry 并返回 `ELOOP` | `libglusterfs/src/inode.c:278` |
| `inode_forget` 传入 0 | 直接把 `nlookup` 归零 | `libglusterfs/src/inode.c:740` |
| 上层欠减 `nlookup` | 记录 critical 日志并纠正计数，不中断服务 | `libglusterfs/src/inode.c:740` |

最后一条来自内核行为异常导致的引用计数不平，代码选择容错而不是断言。

## 并发与锁

两把锁分工明确：

| 锁 | 保护对象 |
| --- | --- |
| `inode_table_t.lock`（互斥量） | 两张哈希表与 active/lru/purge/invalidate 链表 |
| `inode_t.lock` | 该 inode 的 `_ctx` 槽位与 `fd_list` |

`nlookup`、`kids` 是原子计数，可以在不持锁的情况下增减；`ref` 的增减在表锁内进行。以
双下划线开头的函数（`__inode_link`、`__inode_unref`、`__dentry_grep` 等）约定调用方
已经持有表锁，它们自身不加锁。

加锁顺序固定为「先表锁、后 inode 锁」，反向获取会与 fd 的路径冲突并导致死锁，相关说明
见 `libglusterfs/src/fd.c:618` 附近。

## 配置项

| 配置 | 作用范围 | 默认值 | 位置 |
| --- | --- | --- | --- |
| `--lru-limit` | `glusterfs` 进程命令行参数 | 由调用方决定 | `glusterfsd/src/glusterfsd.c:197` |
| `lru-limit` | `mount/fuse` 的 volfile 选项 | 由 volfile 提供 | `xlators/mount/fuse/src/fuse-bridge.c:6884` |
| `inode-lru-limit` | `protocol/server` 的 volfile 选项 | 16384 | `xlators/protocol/server/src/server-helpers.c:540` |

## 与其他模块的交互

| 交互对象 | 形式 | 位置 |
| --- | --- | --- |
| `mount/fuse` | 使用带 invalidator 的表，把回收决策转成内核失效通知 | `xlators/mount/fuse/src/fuse-bridge.c:6450` |
| `protocol/server` | 每个已连接客户端一张表，上限取 `inode-lru-limit` | `xlators/protocol/server/src/server-handshake.c:724` |
| `api`（libgfapi） | 应用进程内自建表，LRU 上限较大 | `api/src/glfs-primary.c:40` |
| `fd` | fd 挂在 `inode->fd_list` 上，两者的生命周期联动 | 见 [fd](fd.md) |
| 自建表的模块 | `features/trash`、`cluster/ec`、`features/bit-rot` 的 bitd、`nfs/server`、`glusterd` 的 quotad、`dht` 的再平衡 | 各模块的 `inode_table_new` 调用 |

## 可观测性

`inode_table_dump`（`libglusterfs/src/inode.c:2427`）输出表级信息：两张哈希表的桶数、
表名、`lru_limit`、`active_size`、`lru_size`、`purge_size`、`invalidate_size`，随后按
链表逐条打印 inode。

单条 inode 的字段由 `inode_dump`（`libglusterfs/src/inode.c:2348`）输出，包括 `gfid`、
`nlookup`、`fd-count`、`active-fd-count`、`ref`、`invalidate-sent`、`ia_type`、
`kids` 与命名空间信息。排查引用泄漏时重点看 `nlookup` 与 `ref` 的组合，以及 ctx 槽位中
记录的归属 xlator。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 类型与接口声明 | `libglusterfs/src/glusterfs/inode.h` |
| 实现 | `libglusterfs/src/inode.c` |
| 哈希与 dentry 工具 | `libglusterfs/src/inode.c:160`、`:177`、`:278`、`:622` |
| 状态迁移 | `libglusterfs/src/inode.c:402`、`:409`、`:427` |
| 淘汰 | `libglusterfs/src/inode.c:1555` |
| statedump | `libglusterfs/src/inode.c:2348`、`:2427` |

## 相关文档

- [fd](fd.md)
- [dict](dict.md)
- [术语表](../overview/glossary.md)
- [调用栈与 FOP 模型](../core/call-stack-and-fop.md)
