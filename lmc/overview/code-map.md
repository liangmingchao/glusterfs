# 代码地图

本文给出「某个能力大致在哪一层、哪个目录」的索引，用于把问题快速缩小到具体文件。模块
内部的实现细节在 [模块索引](../modules/README.md) 对应文档中展开。

## 顶层目录

| 目录 | 职责 | 关键入口 |
| --- | --- | --- |
| `libglusterfs/` | 所有进程共用的核心库：xlator 框架、图、数据结构、日志、事件、同步任务、内存池 | `libglusterfs/src/xlator.c`、`libglusterfs/src/graph.c`、`libglusterfs/src/graph.y` |
| `rpc/` | RPC 框架、传输层与 XDR 定义 | `rpc/rpc-lib/src/rpcsvc.c`、`rpc/rpc-lib/src/rpc-clnt.c`、`rpc/xdr/src/` |
| `api/` | `libgfapi`：不经 FUSE 直接被应用链接的客户端库 | `api/src/glfs.c`、`api/src/glfs-fops.c`、`api/src/glfs-resolve.c` |
| `glusterfsd/` | 主程序：服务端进程、挂载进程、管理进程共同的可执行入口，以及 gf_attach 工具 | `glusterfsd/src/glusterfsd.c`、`glusterfsd/src/glusterfsd-mgmt.c`、`glusterfsd/src/gf_attach.c` |
| `xlators/` | 全部 translator，按用途分类存放 | 见下文「xlators 分类」 |
| `cli/` | `gluster` 命令行：参数解析、命令实现、RPC 客户端、输出格式 | `cli/src/cli.c`、`cli/src/cli-cmd-parser.c`、`cli/src/cli-rpc-ops.c` |
| `heal/` | `glfs-heal` 工具 | `heal/src/glfs-heal.c` |
| `geo-replication/` | 异地复制（gsyncd） | `geo-replication/src/gsyncd.c`、`geo-replication/syncdaemon/` |
| `tools/` | 辅助工具：`glusterfind`、缺失文件查找、`setgfid2path` | `tools/glusterfind/src/main.py`、`tools/gfind_missing_files/gcrawler.c` |
| `events/` | 事件键定义生成器 | `events/eventskeygen.py` |
| `extras/` | 打包脚本、hook 脚本、示例 volfile | `extras/thin-arbiter/thin-arbiter.vol` |
| `contrib/` | 内嵌的第三方与辅助代码 | `contrib/timer-wheel/`、`contrib/userspace-rcu/`、`contrib/xxhash/` |
| `build-aux/` | 构建辅助脚本 | `build-aux/pkg-version` |
| `doc/` | 手册页与开发者文档 | `doc/mount.glusterfs.8`、`doc/developer-guide/` |
| `tests/` | 功能测试与测试框架 | `tests/include.rc`、`run-tests.sh` |

## libglusterfs 关键文件

`libglusterfs` 是理解全部其他代码的前置。按关注点索引：

| 关注点 | 文件 |
| --- | --- |
| translator 的加载、初始化、默认实现填充 | `libglusterfs/src/xlator.c` |
| 图的构建、初始化、激活、重配置 | `libglusterfs/src/graph.c` |
| volfile 语法与解析 | `libglusterfs/src/graph.y`、`libglusterfs/src/graph.l` |
| 请求上下文：调用栈与调用帧 | `libglusterfs/src/glusterfs/stack.h`、`libglusterfs/src/stack.c` |
| FOP 编号与协议可见操作集合 | `libglusterfs/src/glusterfs/glusterfs-fops.h` |
| 默认 fop 实现（构建期生成） | `libglusterfs/src/defaults-tmpl.c`、`libglusterfs/src/gen-defaults.py` |
| fop 参数暂存（call stub 与恢复用） | `libglusterfs/src/default-args.c` |
| 同步化调用封装 | `libglusterfs/src/syncop.c`、`libglusterfs/src/glusterfs/syncop.h` |
| 目录项、inode、fd、属性 | `libglusterfs/src/gf-dirent.c`、`libglusterfs/src/inode.c`、`libglusterfs/src/fd.c`、`libglusterfs/src/glusterfs/iatt.h` |
| 键值容器与序列化 | `libglusterfs/src/dict.c`、`libglusterfs/src/store.c` |
| 日志、事件、状态转储 | `libglusterfs/src/logging.c`、`libglusterfs/src/events.c`、`libglusterfs/src/statedump.c` |
| 卷选项解析与校验 | `libglusterfs/src/options.c`、`libglusterfs/src/glusterfs/options.h` |
| 线程池、定时器、内存池、引用计数 | `libglusterfs/src/event-epoll.c`、`libglusterfs/src/timer.c`、`libglusterfs/src/mem-pool.c`、`libglusterfs/src/refcount.c` |
| 全局常量与路径 | `libglusterfs/src/glusterfs/glusterfs.h` |

## xlators 分类

`xlators/` 下的第一级目录表示用途分类，第二级目录是一个个 translator，其中
`<name>.c` 通常只放能力表与 `xlator_api` 注册，实现分散在同目录的其他文件中。

| 分类 | 包含的 translator | 分类含义 |
| --- | --- | --- |
| `cluster/` | `afr`、`dht`、`ec` | 数据在多个子卷之间的分布与冗余 |
| `protocol/` | `client`、`server`、`auth` | 跨节点通信与认证 |
| `storage/` | `posix` | 落到本地文件系统 |
| `performance/` | `io-cache`、`io-threads`、`md-cache`、`nl-cache`、`open-behind`、`quick-read`、`read-ahead`、`readdir-ahead`、`write-behind` | 缓存、合并、并发调度等性能手段 |
| `features/` | `arbiter`、`barrier`、`bit-rot`、`changelog`、`cloudsync`、`compress`、`gfid-access`、`index`、`leases`、`locks`、`marker`、`metadisp`、`namespace`、`quiesce`、`quota`、`read-only`、`sdfs`、`selinux`、`shard`、`simple-quota`、`snapview-client`、`snapview-server`、`thin-arbiter`、`trash`、`upcall`、`utime` | 面向具体需求的功能 |
| `mgmt/` | `glusterd` | 管理进程，规模上属于独立的子系统 |
| `mount/` | `fuse` | 通过 FUSE 向内核提供文件系统 |
| `nfs/` | `server` | NFS 协议访问 |
| `system/` | `posix-acl` | 与宿主系统语义对接 |
| `meta/` | `meta` | 以虚拟文件形式暴露进程内部状态 |
| `debug/` | `io-stats`、`delay-gen`、`error-gen`、`sink`、`trace` | 观测与故障注入，不属于正常数据路径 |
| `playground/` | `template`、`rot-13` | 教学与示例 |

每个 translator 的能力等级由 `category` 字段声明
（`libglusterfs/src/glusterfs/xlator.h:869`），取值含义见
`gf_category_t`（`libglusterfs/src/glusterfs/glusterfs.h:434`）。例如 `posix` 声明为
`GF_MAINTAINED`（`xlators/storage/posix/src/posix.c:87`），`meta` 声明为
`GF_TECH_PREVIEW`（`xlators/meta/src/meta.c:279`）。

目录名与 volfile 中使用的类型名不一定一致，定位时以 `identifier` 与安装后的模块名为准。
两种常见情形：

- 模块名与目录名不同：`xlators/features/compress/` 构建出的模块是 `cdc`
  （`xlators/features/compress/src/Makefile.am:1`），`identifier` 也是 `cdc`
  （`xlators/features/compress/src/cdc.c:337`），volfile 中写作 `type features/cdc`；
- 一个模块被安装成多个类型名：`xlators/system/posix-acl/` 构建出 `posix-acl.so`
  （`xlators/system/posix-acl/src/Makefile.am:1`），安装时另外创建指向它的软链接
  `features/access-control.so`（`xlators/system/posix-acl/src/Makefile.am:22`），
  因此 volfile 中的 `type features/access-control` 实际加载的是 posix-acl 模块，
  其 `identifier` 也是 `access-control`
  （`xlators/system/posix-acl/src/posix-acl.c:2242`）。

## rpc 目录结构

| 路径 | 内容 |
| --- | --- |
| `rpc/rpc-lib/src/rpcsvc.c` | 服务端 RPC 框架：程序注册、请求分派、认证 |
| `rpc/rpc-lib/src/rpc-clnt.c` | 客户端 RPC 框架：连接管理、请求排队、回复处理 |
| `rpc/rpc-lib/src/rpc-transport.c` | 传输层抽象与动态加载 |
| `rpc/rpc-transport/socket/` | 基于 socket 的具体传输实现 |
| `rpc/xdr/src/*.x` | RPC 载荷的 XDR 定义，构建期生成 `*_xdr.c` |
| `rpc/xdr/src/glusterfs4-xdr.x` | 文件操作相关的 XDR 定义 |

细节见 [RPC 与传输](../core/rpc-and-transport.md)。

## 快速定位方法

几个在排查时最常用的检索起点：

| 想找什么 | 检索方式 |
| --- | --- |
| 某个 option 的取值与默认值 | 在 `xlators/**` 中检索选项名，命中 `volume_options` 数组 |
| 某个 translator 实现了哪些 fop | 打开该模块的 `<name>.c`，查看 `xlator_fops` 结构体初始化 |
| 某个 fop 从应用到落盘经过哪些层 | 从 `xlators/mount/fuse/src/fuse-bridge.c` 或 `api/src/glfs-fops.c` 顺藤向下 |
| 命令对应的处理逻辑 | `cli/src/cli-cmd-parser.c` 找命令解析，`cli/src/cli-rpc-ops.c` 找 RPC 调用，`xlators/mgmt/glusterd/src/glusterd-handler.c` 找服务端处理 |
| 卷配置如何变成 volfile | `xlators/mgmt/glusterd/src/glusterd-volgen.c` |
| 某个内部状态如何观察 | `libglusterfs/src/statedump.c`，以及 `xlators/meta/src/` 下的虚拟文件实现 |

## 相关文档

- [架构总览](architecture.md)
- [模块索引](../modules/README.md)
- [特性索引](../features/README.md)
