# fd

`fd_t` 表示进程内「一个已打开文件的引用」。它的身份由 inode 与打开者（pid）共同决定，
与操作系统文件描述符无关：gluster 内部传递的是 `fd_t` 指针，而给内核或给对端传输的整数
fd 编号由 fdtable 单独维护并映射回 `fd_t`。fd 还是各 xlator 挂载「每个打开实例的私有
状态」的载体。

## 目录

- 概览
- 核心概念
- 数据结构
- 工作流程
- 关键分支与边界条件
- 并发与锁
- 与其他模块的交互
- 可观测性
- 代码位置

## 概览

fd 出现在两类场景：

| 场景 | 说明 |
| --- | --- |
| 面向用户文件的 I/O | `open`、`create`、`readv`、`writev`、`flush`、`fsync` 等 FOP 的第一个参数 |
| 内部无句柄访问 | 配额统计、位衰减、切分、纠删码等需要绕过应用 open 的读写，使用匿名 fd |

## 核心概念

**身份是 (inode, pid)。** `fd_lookup(inode, pid)` 在 inode 的 fd 链表中按 pid 匹配，
pid 传 0 表示「任意一个非匿名 fd」。因此同一个 inode 上可以有多个 fd，分别属于不同的
打开者。

**匿名 fd。** 不对应任何一方的真实 open，用于内部 I/O。它的 pid 为 0，带 `anonymous`
标记，并带一套固定的打开标志（`GF_ANON_FD_FLAGS`，
`libglusterfs/src/glusterfs/fd.h:21`）。同一个 inode 上，相同标志组合的匿名 fd 只会有
一个，被后续请求复用（`fd_anonymous_with_flags`，`libglusterfs/src/fd.c:754`）。

**per-xlator 上下文。** 与 inode 一样，fd 上有一组按 xlator 槽位组织的 `_ctx`，用于
保存「这个打开实例」的私有状态，例如锁上下文、缓存偏移量。`fd->xl_count` 记录已经使用
过的槽位数，注销时按它遍历。

**两个 fd 计数。** inode 上同时维护 `fd_count` 与 `active_fd_count`：前者是曾经打开过的
fd 总数，后者是仍挂在链表上的活跃 fd 数。`fd_destroy` 只在 `bound` 为真时才回退
`fd_count`。

## 数据结构

`fd_t`（`libglusterfs/src/glusterfs/fd.h:38`）的核心字段：

| 字段 | 含义 |
| --- | --- |
| `inode` | 所属文件对象 |
| `pid` | 打开者标识，匿名 fd 为 0 |
| `flags` | 打开标志，匿名 fd 额外叠加 `GF_ANON_FD_FLAGS` |
| `refcount` | 原子引用计数 |
| `inode_list` | 挂入 `inode->fd_list` 的链表节点 |
| `lock` | **仅**用于保护 `_ctx` 数组 |
| `_ctx` | 按 xlator 槽位组织的私有数据数组 |
| `lk_ctx` | 锁上下文，锁操作专用 |
| `xl_count` | 已使用的 ctx 槽位数 |
| `anonymous` | 是否为匿名 fd |

`struct _fd_ctx`（`libglusterfs/src/glusterfs/fd.h:27`）每个槽位保存两组「64 位值或
指针」，第一组的键位置记录归属的 xlator。

`fdtable_t`（`libglusterfs/src/glusterfs/fd.h:61`）解决的是另一个问题：把整数 fd 编号与
`fd_t` 对应起来。它由固定大小的 `fdentries` 数组、空闲链表头 `first_free` 与一把读写锁
组成，条目中的 `next_free` 用 `GF_FDTABLE_END` 与 `GF_FDENTRY_ALLOCATED` 两个哨兵值
标记链表终止与已分配（`libglusterfs/src/glusterfs/fd.h:71`、`:76`）。

fdtable 与 `fd_t` 的区别是本模块最需要分清的一点：`fd_t` 是对象，`fdtable_t` 是编号
分配器，只有在需要把 fd 编号送出去或收进来时才涉及。

## 工作流程

### 创建与绑定

| 步骤 | 接口 | 说明 |
| --- | --- | --- |
| 1 | `fd_allocate`（`libglusterfs/src/fd.c:593`） | 分配对象、初始化锁与引用计数，**不**取 inode 引用 |
| 2 | `fd_create` / `fd_create_uint64`（`libglusterfs/src/fd.c:662`、`:647`） | 在 `fd_allocate` 成功后补取一次 inode 引用 |
| 3 | `fd_bind`（`libglusterfs/src/fd.c:575`） | 挂入 `inode->fd_list`，递增 `fd_count` 与 `active_fd_count` |

创建与绑定分开的原因见 `libglusterfs/src/fd.c:618` 附近的说明：在持有 inode 锁时调用
`inode_ref` 会反向获取 inode 表的锁，与既定的加锁顺序冲突，因此引用由调用方在锁外补取。

### 查找与复用

`fd_lookup`（`libglusterfs/src/fd.c:694`）与 `fd_lookup_uint64`
（`libglusterfs/src/fd.c:714`）都转调 `__fd_lookup`（`libglusterfs/src/fd.c:668`），
行为由 pid 决定：

| pid 取值 | 匹配结果 |
| --- | --- |
| 非 0 | 该 pid 打开的第一个 fd |
| 0 | 第一个非匿名 fd |

匿名 fd 的获取是「有则复用、无则创建」：`fd_anonymous_with_flags`
（`libglusterfs/src/fd.c:754`）先按 (inode, flags) 查已有匿名 fd，未命中才创建并绑定。
`fd_anonymous`（`libglusterfs/src/fd.c:791`）是不带额外标志的简化入口。

服务端有一处典型用法：客户端可能发起不带 fd 的 `read`/`write`（按 GFID 直接读写），
此时服务端生成匿名 fd 来完成解析，见 `resolve_anonfd_simple`
（`xlators/protocol/server/src/server-resolve.c:427`）。

### 引用与销毁

| 阶段 | 接口 | 行为 |
| --- | --- | --- |
| 取引用 | `fd_ref`（`libglusterfs/src/fd.c:427`） | 原子递增 |
| 释放 | `fd_unref`（`libglusterfs/src/fd.c:532`） | 引用计数归零时从 `inode->fd_list` 摘除并递减 `active_fd_count` |
| 销毁 | `fd_destroy`（`libglusterfs/src/fd.c:441`） | 通知各 xlator、释放资源并归还内存 |

`fd_destroy` 的通知顺序是理解 fd 生命周期的关键：它遍历 `_ctx` 中登记过的 xlator，对目录
调用 `releasedir`、对普通文件调用 `release`（`libglusterfs/src/fd.c:441`），因此
「xlator 在 fd 上挂的私有状态」由对应的 `release` 回调负责清理。这一步发生在内存真正释放
之前，之后才销毁锁、释放 `_ctx`、回退 `fd_count`、`inode_unref`、释放锁上下文并归还
内存。

`fd_close`（`libglusterfs/src/fd.c:506`）是另一条路径，用于关闭时通知全图：它遍历
`inode->table->xl->graph->first` 上的所有 xlator，调用其 `fdclose`/`fdclosedir`。它与
`release` 的区别在于触发时机与遍历范围，`close` 对应用户语义，`release` 对应对象销毁。

### 整数 fd 的分配

需要把 fd 编号交给外部时，由 fdtable 负责编号：

| 接口 | 用途 |
| --- | --- |
| `gf_fd_fdtable_alloc` / `gf_fd_fdtable_destroy`（`libglusterfs/src/fd.c:95`、`:198`） | 表的创建与销毁 |
| `gf_fd_unused_get`（`libglusterfs/src/fd.c:238`） | 取一个空闲编号并绑定 `fd_t` |
| `gf_fd_fdptr_get`（`libglusterfs/src/fd.c:388`） | 由编号反查 `fd_t` |
| `gf_fd_put`（`libglusterfs/src/fd.c:293`） | 归还编号 |

两条实际使用路径：FUSE 把编号回给内核（`xlators/mount/fuse/src/fuse-bridge.c:2836`），
服务端把编号放进应答交给客户端（`xlators/protocol/server/src/server-common.c:234`）。

## 关键分支与边界条件

| 条件 | 行为 | 位置 |
| --- | --- | --- |
| 在 inode 上查找时遇到匿名 fd | 跳过，不会被 `fd_lookup` 返回 | `libglusterfs/src/fd.c:668` |
| 匿名 fd 的标志 | 强制叠加 `GF_ANON_FD_FLAGS`，并保留调用方传入的 `O_DIRECT` | `libglusterfs/src/fd.c:754` |
| 引用计数未归零 | 不销毁、不摘链，仅递减计数 | `libglusterfs/src/fd.c:532` |
| `fd_destroy` 时 `fd->inode` 为空或 `_ctx` 为空 | 记录错误并直接返回，不做后续释放 | `libglusterfs/src/fd.c:441` |
| `ctx` 槽位不足 | 动态扩容到 `xl_count + graph->xl_count` | `libglusterfs/src/fd.c:836` |

最后一条是 fd 与 inode 的一处明显差异：inode 的 ctx 数组在表创建时按图中 xlator 总数
一次性分配，而 fd 的 ctx 数组按需增长，见 [inode 与 inode 表](inode.md#数据结构)。

## 并发与锁

| 保护对象 | 锁 | 说明 |
| --- | --- | --- |
| `fd->_ctx` | `fd->lock` | 头文件注释明确该锁**仅**用于 ctx 数组 |
| `inode->fd_list` 与两个 fd 计数 | `inode->lock` | `fd_bind`、`fd_unref`、`__fd_lookup` 都在其保护下操作 |
| `fd->refcount` | 原子操作 | 无需额外加锁 |
| fdtable | `pthread_rwlock_t` | 编号分配与查找并发安全 |

以双下划线开头的函数（`__fd_ctx_set`、`__fd_lookup`、`__fd_bind` 等）约定调用方已持有
对应锁。`fd->lock` 只保护 ctx，不要用它来保护其他字段。

## 与其他模块的交互

| 交互对象 | 形式 | 位置 |
| --- | --- | --- |
| `mount/fuse` | 用 fdtable 分配内核可见的 fd 编号 | `xlators/mount/fuse/src/fuse-bridge.c:2836` |
| `protocol/server` | 用 fdtable 分配发给客户端的 fd 编号，并为无 fd 请求建匿名 fd | `xlators/protocol/server/src/server-common.c:234`、`xlators/protocol/server/src/server-resolve.c:427` |
| `features/quota`、`features/marker` | 用匿名 fd 做内部统计读写 | 各模块的 `fd_anonymous` 调用 |
| `features/shard`、`cluster/ec`、`features/bit-rot` | 用匿名 fd 访问分片或签名的实际文件 | 各模块的 `fd_anonymous` 调用 |
| inode | fd 挂在 inode 上，inode 的 `fd_count`/`active_fd_count` 反映打开情况 | [inode 与 inode 表](inode.md) |

## 可观测性

`fd_dump`（`libglusterfs/src/fd.c:976`）输出单个 fd 的 `pid`、`refcount`、`flags`，
并递归打印其 inode（`inode_dump`）；`fdentry_dump`
（`libglusterfs/src/fd.c:995`）用于打印 fdtable 中的条目。排查泄漏时先看 inode 上的
`fd-count` 与 `active-fd-count` 是否随操作回落，再看对应 `fd_t` 的 `refcount`。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 类型与接口声明 | `libglusterfs/src/glusterfs/fd.h` |
| 实现 | `libglusterfs/src/fd.c` |
| 对象创建与绑定 | `libglusterfs/src/fd.c:593`、`:575` |
| 查找与匿名 fd | `libglusterfs/src/fd.c:668`、`:754` |
| 销毁与释放回调 | `libglusterfs/src/fd.c:441`、`:506` |
| ctx 读写 | `libglusterfs/src/fd.c:836`、`:911` |
| 编号表 | `libglusterfs/src/fd.c:95`、`:238` |

## 相关文档

- [inode 与 inode 表](inode.md)
- [dict](dict.md)
- [调用栈与 FOP 模型](../core/call-stack-and-fop.md)
- [术语表](../overview/glossary.md)
