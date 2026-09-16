# snapview-client

`snapview-client` 插在挂载点客户端图的上层，负责把「普通卷路径」与「快照路径」分开：
普通请求原样交给第一子卷，落在快照入口目录里的请求交给第二子卷（即连到 `snapd` 的
客户端）。同时它承担快照世界的只读语义，对虚拟 inode 的写操作直接返回 `EROFS`。
服务端一侧见 [snapview-server](snapview-server.md)，整体语义见 [快照](../features/snapshot.md)。

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

该 xlator 有两个子卷，顺序有严格约定：

| 子卷 | 内容 | 承载的请求 |
| --- | --- | --- |
| 第一子卷（`FIRST_CHILD`） | 源卷的正常客户端图 | 普通路径的全部 FOP |
| 第二子卷（`SECOND_CHILD`） | 到 `snapd` 的 `protocol/client` | 快照世界中的读操作 |

子卷顺序由卷生成阶段决定（`volgen_graph_build_snapview_client`，
`xlators/mgmt/glusterd/src/glusterd-volgen.c:3470`），注释中明确写了「普通卷路径走
FIRST_CHILD，快照世界走 SECOND_CHILD」这一约定，因此移动这两条连线的顺序会直接破坏
语义。选择逻辑见 `svc_get_subvolume`
（`xlators/features/snapview-client/src/snapview-client.c:28`）。

## 核心概念

**入口点（entry point）。** 挂载点下用于进入快照世界的目录，名字默认为 `.snaps`，可由
卷选项 `features.snapshot-directory` 修改。它不是真实目录，而是由本模块在 `lookup` 与
`readdir` 上「造」出来的。

**inode 类型。** 每个 inode 在本模块的上下文中被标记为
`NORMAL_INODE` 或 `VIRTUAL_INODE`（`xlators/features/snapview-client/src/snapview-client.h:94`），
决定后续 FOP 该走哪个子卷。这一标记在 `lookup` 成功时写入。

**跨层标记。** 进入快照世界的请求会带上 `entry-point` 标记
（`SVC_ENTRY_POINT_SET`，`xlators/features/snapview-client/src/snapview-client.h:41`），
服务端据此区分「入口目录本身」与「入口内的普通目录」。

## 数据结构

| 结构 | 位置 | 作用 |
| --- | --- | --- |
| `svc_local_t` | `xlators/features/snapview-client/src/snapview-client.h:20` | 单次 FOP 的局部数据：目标子卷、loc、fd、回调等 |
| `svc_private_t` | `xlators/features/snapview-client/src/snapview-client.h:79` | 模块私有状态：入口目录名、samba 专用目录、是否显示入口、保护入口名的锁 |
| `svc_fd_t` | `xlators/features/snapview-client/src/snapview-client.h:87` | 每个 fd 的状态：读目录偏移、入口点是否已处理、是否位于 samba 专用目录 |

inode 上的类型标记保存在 inode 上下文中（`svc_inode_ctx_get`/`svc_inode_ctx_set`），
fd 上的状态保存在 fd 上下文中（`svc_fd_ctx_get_or_new`），两者的存储机制见
[inode 与 inode 表](inode.md) 与 [fd](fd.md)。

## 工作流程

### 请求路由

`lookup`（`xlators/features/snapview-client/src/snapview-client.c:369`）是路由的起点：

1. 读取父目录的 inode 类型与自身是否已有类型标记；
2. 名字等于入口点名、或父目录已是虚拟 inode 时，把请求交给第二子卷，并在 `xdata` 中置
   `entry-point`；
3. 否则交给第一子卷；
4. 根据实际负责的子卷，把结果记录为 `NORMAL_INODE` 或 `VIRTUAL_INODE`。

标记完成后，后续 FOP 只需读上下文即可决定去向，无需重复判断路径
（`SVC_GET_SUBVOL_FROM_CTX` 宏，`xlators/features/snapview-client/src/snapview-client.h:63`）。

### 只读语义

所有写类 FOP（`create`、`mkdir`、`mknod`、`symlink`、`link`、`unlink`、`rmdir`、
`setattr`、`setxattr`、`removexattr` 等）在入口点或虚拟 inode 上直接以 `EROFS` 失败，
不向下传递。判定通常是「父目录不是 `NORMAL_INODE`，或名字就是入口点」，实现样例见
`gf_svc_create`（`xlators/features/snapview-client/src/snapview-client.c:1326`）。

### 目录读取

`readdir` 与 `readdirp` 的特殊之处在于：当用户读取被配置为入口展示位置的非根目录时，
模块需要把入口条目「插」进结果里，这由 `svc_fd_t` 的偏移与 `entry_point_handled` 标记
控制，相关实现见 `gf_svc_readdir`
（`xlators/features/snapview-client/src/snapview-client.c:1643`）与 `gf_svc_readdirp`
（`xlators/features/snapview-client/src/snapview-client.c:2086`）。

## 关键分支与边界条件

| 条件 | 行为 | 位置 |
| --- | --- | --- |
| 客户端刚上线，inode 上下文缺失 | 先按普通路径处理；若返回 `ENOENT`/`ESTALE` 再改投快照世界 | `xlators/features/snapview-client/src/snapview-client.c:296` |
| 卷文件变更导致上下文丢失 | 同上，靠 `ESTALE` 回退 | 同上 |
| 在虚拟 inode 上执行写操作 | 直接返回 `EROFS`，不 wind | 各写类 FOP 入口 |
| 跨世界的 `rename`/`link` | 拒绝，避免把快照对象带入源卷或反之 | `xlators/features/snapview-client/src/snapview-client.c:2156` 附近的说明 |
| `snapdir-entry-path` 与入口目录重名 | 由选项校验拒绝 | `validate_uss_dir`，`xlators/mgmt/glusterd/src/glusterd-volume-set.c:146` |

## 并发与锁

入口目录名保存在 `svc_private_t.path` 中，重配置会改写它，因此读写都要在
`svc_private_t.lock` 保护下进行。inode 类型与 fd 状态使用框架的上下文 API，由 inode 锁
与 fd 锁保护。

## 配置项

| 选项 | 含义 | 默认值 | 对应卷选项 |
| --- | --- | --- | --- |
| `snapshot-directory` | 快照入口目录名 | `.snaps` | `features.snapshot-directory` |
| `snapdir-entry-path` | 需要展示入口条目的目录（供 Samba 使用） | 空 | `features.snapdir-entry-path` |
| `show-snapshot-directory` | 是否在 `snapdir-entry-path` 的 `readdir` 结果中显示入口 | off | `features.show-snapshot-directory` |

卷选项与 xlator 选项的映射表在
`xlators/mgmt/glusterd/src/glusterd-volume-set.c:1834` 附近。其中 `snapdir-entry-path`
不可重配置（`xlators/features/snapview-client/src/snapview-client.c:2469` 附近的注释）。

## 与其他模块的交互

| 交互对象 | 形式 |
| --- | --- |
| 源卷的客户端图 | 作为第一子卷，承接全部普通请求 |
| `snapd` | 作为第二子卷，经 `protocol/client` 访问，服务端由 [snapview-server](snapview-server.md) 处理 |
| `glusterd` | 由卷生成逻辑决定是否插入本模块、以及两个子卷分别接什么 |
| `mount/fuse` 或 `libgfapi` | 作为上层调用者，`.snaps` 的可见性最终体现在它们的 `readdir` 与 `lookup` 结果中 |

## 可观测性

路由决策与只读拒绝都会写日志：正常图的 `lookup` 失败按 `ENOENT`/`ESTALE` 降级为调试
级别，虚拟图失败按错误级别记录
（`xlators/features/snapview-client/src/snapview-client.c:315`），因此排查
「看不到快照」类问题时先看这里的日志，再确认 `snapd` 是否在运行、快照是否已激活。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 模块注册与能力表 | `xlators/features/snapview-client/src/snapview-client.c:2743`、`:2769` |
| 路由与只读实现 | `xlators/features/snapview-client/src/snapview-client.c` |
| 数据结构与上下文 | `xlators/features/snapview-client/src/snapview-client.h` |
| 图生成 | `xlators/mgmt/glusterd/src/glusterd-volgen.c:3470` |
| 卷选项映射 | `xlators/mgmt/glusterd/src/glusterd-volume-set.c:1834` |

## 相关文档

- [snapview-server](snapview-server.md)
- [快照](../features/snapshot.md)
- [调用栈与 FOP 模型](../core/call-stack-and-fop.md)
- [inode 与 inode 表](inode.md)
