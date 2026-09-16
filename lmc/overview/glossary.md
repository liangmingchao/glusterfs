# 术语表

本表定义知识库中统一使用的术语。写作时先查本表，新增术语先登记再使用；同一概念在全部
文档中只使用这里给出的名称。

## 目录

- 核心对象
- 请求处理
- 命名与标识
- 卷类型与数据组织
- 进程与运维

## 核心对象

**xlator**（translator，翻译器/转换器）
组成 gluster 卷处理逻辑的模块单元。每个 xlator 是一个共享库，导出唯一的 `xlator_api`
符号，声明自己的初始化函数、能力表与选项集合。类型定义见
`xlator_api_t`（`libglusterfs/src/glusterfs/xlator.h:854`），运行时实例见 `xlator_t`
（`libglusterfs/src/glusterfs/xlator.h:750`）。中文行文中保留英文原词，不写作「翻译器」。

**graph**
一棵由 xlator 组成的树，是卷在一个进程内的完整形态。请求只能在树内上下流动，无法跨
分支。类型为 `glusterfs_graph_t`（`libglusterfs/src/glusterfs/glusterfs.h:625`）。

**volfile**
描述一棵 graph 的文本文件，声明每个节点的 `type`、`name`、`option` 和 `subvolumes`。
既可以手工编写用于调试，也由 `glusterd` 根据卷配置自动生成。

**subvolume**
graph 中某个节点的下级节点集合。同一条 volfile 里，`subvolumes` 声明的是「这个 xlator
的下级是谁」。

**brick**
一个卷在某个节点上的一份本地存储目录及其支撑进程。每个 brick 对应一个独立的服务端
graph 和一条独立的进程（或进程内的一个图），其顶层节点类型为 `storage/posix`。

**client_t**
服务端对「一个已连接客户端」的抽象，保存该客户端的身份、认证信息与锁相关的状态。
定义见 `libglusterfs/src/glusterfs/client_t.h:35`。

## 请求处理

**FOP**（file operation）
translator 之间传递的一次文件操作，例如 `lookup`、`readv`、`writev`、`setxattr`。编号
集合定义在 `libglusterfs/src/glusterfs/glusterfs-fops.h`，能力表见
`struct xlator_fops`（`libglusterfs/src/glusterfs/xlator.h:546`）。

**wind / unwind**
请求向下传递给子 xlator 称为 wind，应答向上返回给父 xlator 称为 unwind，对应宏
`STACK_WIND`（`libglusterfs/src/glusterfs/stack.h:283`）与
`STACK_UNWIND_STRICT`（`libglusterfs/src/glusterfs/stack.h:346`）。

**call frame**（调用帧）
一次请求在某一层 xlator 内的执行上下文，携带 `this`、`parent`、`cookie`、`local` 等
信息，类型为 `call_frame_t`（`libglusterfs/src/glusterfs/stack.h:58`）。

**call stack**（调用栈）
一个请求从进入 graph 到离开所经过的全部帧，类型为 `call_stack_t`
（`libglusterfs/src/glusterfs/stack.h:87`）。一次请求对应一个调用栈。

**syncop / synctask**
把异步的 FOP 调用改写成同步写法的机制：`synctask` 是一个可以在等待 FOP 结果时让出
的轻量执行单元。实现见 `libglusterfs/src/syncop.c`。

**notify**
xlator 之间传递的状态事件通道，用于上下级通报「子卷就绪」「连接断开」等变化，与 FOP
的数据通路相互独立。事件集合定义在 `libglusterfs/src/glusterfs/events.h`。

## 命名与标识

**GFID**（gluster file identifier）
文件和目录在卷内的全局唯一标识，128 位 UUID，以扩展属性形式持久化，键名为
`trusted.gfid`（`libglusterfs/src/glusterfs/glusterfs.h:135`）。路径会变，GFID 不变，
因此内部引用与自愈都以 GFID 为准。

**pgfid**
父目录的 GFID，用于在目录内定位一个条目，键名前缀为 `trusted.pgfid.`
（`libglusterfs/src/glusterfs/glusterfs.h:136`）。

**handle**
brick 上以 GFID 命名的硬链接路径，形如 `<brick>/.glusterfs/<前两位>/<后两位>/<GFID>`。
它把「可变的用户路径」与「不变的 GFID」解耦，brick 内部操作走 handle。
路径拼接见 `MAKE_HANDLE_ABSPATH`（`xlators/storage/posix/src/posix-handle.h:139`），
其中隐藏目录名由 `GF_HIDDEN_PATH` 定义
（`libglusterfs/src/glusterfs/common-utils.h:478`）。

**iatt**
一次 FOP 返回的属性集合，包含 `st_ino`、`ia_ino`、`ia_gfid`、`ia_type`、`ia_size`、
时间戳等，是 gluster 内部传递属性的统一结构，定义见
`libglusterfs/src/glusterfs/iatt.h:46`。

**loc_t**
一次操作的定位信息，组合「父目录 inode + 名字 + GFID + 路径」，定义见
`struct _loc`（`libglusterfs/src/glusterfs/xlator.h:59`）。

**inode / inode table**
gluster 内部对文件对象的引用，以 GFID 为键，由 inode table 统一管理，作用类似内核
inode 缓存。实现见 `libglusterfs/src/inode.c`。

**fd**
对已打开文件的引用，定义见 `libglusterfs/src/glusterfs/fd.h`。`fd_t` 与 inode 分离，
同一个 inode 可以有多个 fd。

**dict / xdata**
`dict_t` 是 gluster 内部的键值容器（`libglusterfs/src/dict.c`）。随 FOP 一起传递的
`dict` 参数统一称为 xdata，用于携带不属于标准参数的附加信息。

**op_version**
xlator 能力版本。每个 xlator 在 `xlator_api_t` 中用 `op_version` 声明自身从哪个版本
起可用（`libglusterfs/src/glusterfs/xlator.h:859`），卷生成时据此决定是否把该节点放入
graph，集群版本槽位数为 `GF_MAX_RELEASES`（`libglusterfs/src/glusterfs/options.h:85`）。

## 卷类型与数据组织

**DHT**（Distributed Hash Table）
负责把目录与文件分配到不同子卷的 xlator，是「无元数据服务器」得以成立的基础。见
`xlators/cluster/dht/`。

**AFR**（Automatic File Replication）
副本与自愈机制，负责在多个子卷之间保持副本一致。见 `xlators/cluster/afr/`。

**EC**（Erasure Coding）
纠删码机制，以编码分片替代完整副本。见 `xlators/cluster/ec/`。

**arbiter**
三副本场景下把第三份替换为只存元数据的仲裁副本，用于在保证一致性的同时节省空间。
见 `xlators/features/arbiter/`。

**thin-arbiter**
把仲裁角色从数据节点中剥离，放到一个独立轻量节点上的仲裁方案。见
`xlators/features/thin-arbiter/` 和示例 volfile `extras/thin-arbiter/thin-arbiter.vol`。

**split-brain**（脑裂）
副本之间内容冲突且无法自动判定哪一份正确，需要人工介入的状态。

**self-heal**（自愈）
把缺失或过期的副本补齐到一致状态的过程，由 `glustershd` 或按需触发的扫描执行。

**rebalance**（再平衡）
子卷集合变化后，重新分布已有文件以恢复负载均衡的过程。

**快照**（snapshot）
对整卷全部 brick 的后端存储做一次时间点副本，并把副本组织成只读卷的能力。粒度是卷，
不是文件或目录。原理与流程见 [快照](../features/snapshot.md)。

**快照卷**（snap volume）
由源卷 volinfo 复制并改写而来的卷对象，其 brick 指向后端副本路径。它与普通卷一样有
volfile、brick 进程与状态，默认创建后处于停止状态，需要激活才对外服务。

**快照后端**（snapshot backend）
提供时间点副本能力的具体实现，通过 `glusterd_snap_ops` 接口接入。当前实现为精简置备的
LVM 逻辑卷与 ZFS dataset 两种，见 [快照](../features/snapshot.md#实现原理)。

## 进程与运维

**glusterd**
管理进程，保存集群拓扑与卷配置，生成 volfile，并负责拉起与重配其他进程。

**brick 进程**
承载某个 brick 服务端 graph 的进程，由 `glusterd` 启动。

**glustershd**
自愈守护进程，承载卷的自愈 graph。

**snapd**
用户自助访问快照的服务进程，每个开启该能力的卷一个，进程内承载 `features/snapview-server`，
按需为快照建立 `libgfapi` 客户端实例。见 [snapview-server](../modules/snapview-server.md)。

**USS**（User Serviceable Snapshots）
「用户自助访问快照」能力的缩写，通过卷选项 `features.uss` 开启。开启后挂载点下出现
`.snaps` 入口目录，用户可自行浏览只读快照，无需管理员逐个激活与挂载。见
[snapview-client](../modules/snapview-client.md)。

**barrier**（I/O 屏障）
在需要全局一致时间点的操作（如快照创建）期间阻塞应用 I/O 的机制。开启与关闭通过下发到
brick 的 barrier 请求完成，见 [快照](../features/snapshot.md#一致性与异常处理)。

**pass-through**
xlator 的一种运行状态：被标记为 pass-through 时，未显式实现或按配置需要跳过的 FOP
直接转发给子节点，不参与处理。相关字段为 `pass_through`
（`libglusterfs/src/glusterfs/xlator.h:801`）与 `pass_through_fops`
（`libglusterfs/src/glusterfs/xlator.h:802`）。

**statedump**
进程内部状态的文本转储，包含各 xlator 的私有结构、内存与锁信息，用于故障分析。实现见
`libglusterfs/src/statedump.c`。

**meta xlator**
把进程内部状态以虚拟文件形式暴露出来的 xlator，路径通常位于运行目录下的 `meta`
子目录。见 `xlators/meta/src/meta.c`。

## 相关文档

- [架构总览](architecture.md)
- [代码地图](code-map.md)
- [xlator 框架](../core/xlator-framework.md)
