# snapview-server

`snapview-server` 运行在 `snapd` 进程里，把一个卷的已激活快照以只读方式暴露给挂载点。
它自身不读磁盘：每遇到一个需要访问的快照，就用 `libgfapi` 在进程内建立一个该快照卷的
客户端实例，把上游传来的 `lookup`、`readdir`、`open`、`readv` 等请求转成对快照卷的
调用。因此它是「快照世界」的服务端入口，与 [snapview-client](snapview-client.md) 成对
工作，整体语义见 [快照](../features/snapshot.md)。

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

在 `snapd` 进程的图中，`snapview-server` 位于数据通路的末端，客户端请求经
`protocol/server` → `debug/io-stats` → `performance/io-threads` 到达它
（图由 `glusterd_snapdsvc_generate_volfile` 生成，
`xlators/mgmt/glusterd/src/glusterd-volgen.c:5858`）。它没有子 xlator，取而代之的是进程内
的 `glfs` 实例。

它只实现读操作，`xlator_fops` 表中不含任何写入类 FOP
（`xlators/features/snapview-server/src/snapview-server.c:2686`），这是快照只读语义的第一
道保证；第二道在客户端侧的 [snapview-client](snapview-client.md)。

## 核心概念

**入口点与三种 inode。** 快照世界里的 inode 被分为三类，定义在
`inode_type_t`（`xlators/features/snapview-server/src/snapview-server.h:129`）：

| 类型 | 含义 |
| --- | --- |
| `SNAP_VIEW_ENTRY_POINT_INODE` | 入口目录本身（挂载点下的 `.snaps`） |
| `SNAP_VIEW_SNAPSHOT_INODE` | 某个快照的根目录 |
| `SNAP_VIEW_VIRTUAL_INODE` | 快照内的文件与目录 |

**合成 GFID。** 快照里的对象在客户端需要有稳定的标识，但又不能与源卷对象冲突。
`svs_uuid_generate`（`xlators/features/snapview-server/src/snapview-server-helpers.c:334`）
用 `xxh64(快照名 + 原始 GFID)` 生成合成 GFID，因此同一文件在同一快照内始终得到同一个
标识，不同快照或与源卷之间则互不相同。

**句柄映射。** 每个虚拟 inode 在 `svs_inode_t` 中保存一对 `(glfs_t *, glfs_object_t *)`，
指向它在快照卷中的真实对象；打开的文件在 `svs_fd_t` 中保存 `glfs_fd_t *`。

## 数据结构

| 结构 | 位置 | 作用 |
| --- | --- | --- |
| `struct svs_inode` | `xlators/features/snapview-server/src/snapview-server.h:135` | 虚拟 inode 的 `glfs` 实例与对象句柄、类型、入口点的父 GFID、快照名 |
| `struct svs_fd` | `xlators/features/snapview-server/src/snapview-server.h:153` | 打开的 `glfs_fd_t` |
| `struct snap_dirent` | `xlators/features/snapview-server/src/snapview-server.h:158` | 一个快照的登记项：名字、UUID、快照卷名、`glfs` 实例 |
| `struct svs_private` | `xlators/features/snapview-server/src/snapview-server.h:166` | 模块私有状态：快照登记数组、数量、卷名、链表、锁、到 `glusterd` 的 RPC 客户端 |

`snap_dirent.fs` 是延迟建立的：只有真正访问某个快照时才初始化，这让未访问的快照不占用
`glfs` 实例。

## 工作流程

### 启动与快照列表

`init`（`xlators/features/snapview-server/src/snapview-server.c:2584`）做两件事：建立到
`glusterd` 的管理 RPC（`svs_mgmt_init`，
`xlators/features/snapview-server/src/snapview-server-mgmt.c:80`），随后拉取初始快照列表
（`svs_get_snapshot_list`，
`xlators/features/snapview-server/src/snapview-server-mgmt.c:445`）。列表随快照的增删动态
更新，避免每次访问都去问 `glusterd`。

### 建立快照访问实例

访问某个快照时按需建立 `glfs` 实例（`__svs_initialise_snapshot_volume`，
`xlators/features/snapview-server/src/snapview-server-helpers.c:450`）：

1. 在快照列表中找到对应的登记项；
2. 用 `glfs_new("/snaps/<快照名>/<快照卷名>/<快照卷名>")` 创建客户端实例，该 volfile id
   由 `glusterd` 特殊解析为快照卷的客户端 volfile；
3. 用 `glfs_set_volfile_server(fs, "tcp", <volfile 服务端>, 24007)` 指定配置来源；
4. `glfs_init` 完成挂载，句柄保存在登记项的 `fs` 字段中供后续复用。

### 请求转换

各 FOP 把参数翻译成 `glfs` 的高层句柄接口，再转换回 gluster 的 `iatt` 与 `dict` 结构返回
上游。以读目录为例，`svs_readdirp` 调用 `glfs_h_opendir`、`glfs_readdirplus_r`，并用
合成 GFID 填充每个条目的标识。

## 关键分支与边界条件

| 条件 | 行为 | 位置 |
| --- | --- | --- |
| 快照未激活 | 不出现在列表中，访问返回失败 | 见 [快照](../features/snapshot.md) 的激活章节 |
| 请求带 `entry-point` 标记 | 按入口点语义处理，构造快照根目录 | `xlators/features/snapview-server/src/snapview-server.c:633` |
| 缓存中的 `glfs` 实例已失效 | 校验后置空并重新建立句柄 | `SVS_CHECK_VALID_SNAPSHOT_HANDLE`，`xlators/features/snapview-server/src/snapview-server.h:42` |
| `glfs` 实例数超上限 | 上限 256（`SNAP_VIEW_MAX_GLFS_T`） | `xlators/features/snapview-server/src/snapview-server.h:32` |
| 打开句柄数超上限 | 上限 1024（`SNAP_VIEW_MAX_GLFS_FDS`） | `xlators/features/snapview-server/src/snapview-server.h:33` |
| 对象句柄数超上限 | 上限 1024（`SNAP_VIEW_MAX_GLFS_OBJ_HANDLES`） | `xlators/features/snapview-server/src/snapview-server.h:34` |

## 并发与锁

快照登记表与其中的 `glfs` 实例由 `svs_private.snaplist_lock` 保护，读取登记项、校验实例
有效性、在动态更新时增删条目都要在该锁内进行。inode 与 fd 上的私有数据使用框架提供的
上下文机制，由 inode 表锁与 fd 锁保护（见 [inode 与 inode 表](inode.md)、[fd](fd.md)）。

## 配置项

| 选项 | 类型 | 含义 |
| --- | --- | --- |
| `volname` | 字符串 | 该 `snapd` 进程服务的卷名，由 `glusterd` 在生成 volfile 时固定写入 |

## 与其他模块的交互

| 交互对象 | 形式 |
| --- | --- |
| `glusterd` | 通过管理 RPC 获取快照列表与快照卷句柄，并接收快照增删通知 |
| `libgfapi` | 进程内为每个快照建立一个 `glfs` 实例，实际的数据读写都经它完成 |
| `protocol/server` | 作为 `snapd` 图的下级，承接来自挂载点的请求 |
| `snapview-client` | 客户端侧的配对模块，负责把 `.snaps` 路径的请求送到本模块 |

## 可观测性

组件自身的状态可以通过日志观察：快照列表的建立与刷新、`glfs` 实例的创建与失效、句柄
上限触达都会记录日志。与其他 translator 一样，任何一次 `lookup` 或 `readdir` 都能通过
`debug/io-stats` 在 `snapd` 图中定位到调用量。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 模块注册与能力表 | `xlators/features/snapview-server/src/snapview-server.c:2686`、`:2717` |
| 请求处理 | `xlators/features/snapview-server/src/snapview-server.c` |
| 数据结构与上下文 | `xlators/features/snapview-server/src/snapview-server.h` |
| `glfs` 实例与句柄管理 | `xlators/features/snapview-server/src/snapview-server-helpers.c` |
| 与 `glusterd` 的交互 | `xlators/features/snapview-server/src/snapview-server-mgmt.c` |
| `snapd` 进程与图 | `xlators/mgmt/glusterd/src/glusterd-snapd-svc.c`、`xlators/mgmt/glusterd/src/glusterd-volgen.c:5858` |

## 相关文档

- [snapview-client](snapview-client.md)
- [快照](../features/snapshot.md)
- [volfile 与图构建](../core/volfile-and-graph.md)
- [fd](fd.md)
