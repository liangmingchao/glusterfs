# volfile 与图构建

volfile 是描述一棵 xlator 树的纯文本配置，决定了某个进程为哪个卷、以什么顺序、带什么
参数加载哪些 xlator。它既可以由 `glusterd` 根据卷配置自动生成，也可以手工编写用于
调试。本文说明 volfile 的语法、从文本到可运行图的构建过程，以及配置变更如何生效。
树中节点之间的请求传递见 [调用栈与 FOP 模型](call-stack-and-fop.md)。

## 语法

一个 volfile 由若干个 `volume` 块顺序构成，每个块声明一个 xlator 实例：

```text
volume <实例名>
    type <xlator 类型>
    option <键> <值>
    subvolumes <子实例名> [<子实例名> ...]
end-volume
```

关键字含义：

| 关键字 | 必需 | 含义 |
| --- | --- | --- |
| `volume` / `end-volume` | 必需 | 定义块的起止，参数是该实例在树中的名字 |
| `type` | 必需 | xlator 类型，按 `<分类>/<名称>` 或 `<名称>` 写作，加载时对应到同名共享库 |
| `option` | 可选 | 传给该 xlator 的配置，键名取自其 `volume_options` 表 |
| `subvolumes` | 除叶子外必需 | 该实例的下级实例列表，可以有多个 |

其他约定：以 `#` 开头的行是注释；取值范围与默认值由对应 xlator 的选项表定义；文件中出现
的反引号内容会在解析前作为命令执行并替换为输出，这一步由 `preprocess`
（`libglusterfs/src/graph.y` 中的解析前处理）完成。

一个可运行的完整例子是 thin-arbiter 的示例 volfile
（`extras/thin-arbiter/thin-arbiter.vol`），其结构与含义为：存储层 `ta-posix`
（`storage/posix`）在最下方，其上依次叠加 `ta-thin-arbiter`、`ta-locks`、`ta-upcall`、
`ta-io-threads`、`ta-index`，顶层是 `protocol/server`，并把 `/mnt/thin-arbiter` 作为
对外暴露的卷名。

## 从文本到可运行的图

构建分三步，前两步在解析阶段完成，第三步让整棵树开始工作：

| 阶段 | 位置 | 行为 |
| --- | --- | --- |
| 解析 | `libglusterfs/src/graph.y:556`（`glusterfs_graph_construct`） | 逐块读取，创建 xlator 实例，记录类型、选项与依赖关系 |
| 准备 | `libglusterfs/src/graph.c:590`（`glusterfs_graph_prepare`） | 加载各 xlator 模块、连接父子关系、校验选项 |
| 激活 | `libglusterfs/src/graph.c:805`（`glusterfs_graph_activate`） | 自底向上初始化并对外生效 |

父子连接在准备阶段建立，相关函数为 `glusterfs_xlator_link`
（`libglusterfs/src/graph.c:183`）与确定树顶的 `glusterfs_graph_set_first`
（`libglusterfs/src/graph.c:216`）。`glusterfs_graph_init`
（`libglusterfs/src/graph.c:456`）负责调用各节点的 `init`。

## volfile 的来源

**手工编写。** 用于调试单个 xlator 或复现问题，直接作为 `glusterfs` 命令的输入，例子见
`extras/thin-arbiter/thin-arbiter.vol` 与 `tests/features/volspec/`。

**由 glusterd 生成。** 这是生产环境的常态。生成入口是 `build_graph_generic`
（`xlators/mgmt/glusterd/src/glusterd-volgen.c:1070`），它按卷类型与已启用的选项逐层
叠加 xlator 节点，最终输出客户端、brick、自愈等若干份 volfile。

| 产物 | 命名规则 | 位置 |
| --- | --- | --- |
| brick volfile | `<卷名>.<主机名>.<brick 路径>.<后缀>.vol` | `xlators/mgmt/glusterd/src/glusterd-volgen.c:5192`（`get_brick_filepath`） |
| FUSE 客户端 volfile | `<卷名>.tcp-fuse.vol` | `xlators/mgmt/glusterd/src/glusterd-volgen.c:5745` |
| 客户端图生成入口 | — | `xlators/mgmt/glusterd/src/glusterd-volgen.c:5765`（`generate_client_volfiles`） |

所有 volfile 都存放在卷目录下，卷目录名为 `vols` 前缀
（`xlators/mgmt/glusterd/src/glusterd-store.h:28`），其父目录是 `glusterd` 的工作目录
`GLUSTERD_DEFAULT_WORKDIR`（`libglusterfs/src/glusterfs/glusterfs.h:284`）。

服务进程启动时通过 `--volfile-id` 指定使用哪一份 volfile，命令行由
`glusterd_svc_start` 拼装（`xlators/mgmt/glusterd/src/glusterd-svc-mgmt.c:144`）。

## 进程如何取得 volfile

进程自身不读磁盘上的卷配置，而是向 `glusterd` 请求。这条路径在协议上属于握手而非文件
操作：

| 角色 | 位置 |
| --- | --- |
| 请求方发起 GETSPEC | `glusterfsd/src/glusterfsd-mgmt.c:2532` |
| 请求方接收应答 | `glusterfsd/src/glusterfsd-mgmt.c:2275`（`mgmt_getspec_cbk`） |
| glusterd 侧处理 | `xlators/mgmt/glusterd/src/glusterd-handshake.c:867`（`__server_getspec`） |
| glusterd 侧分派表 | `xlators/mgmt/glusterd/src/glusterd-handshake.c:1789` |

应答中除了 volfile 正文还带有校验和，客户端把它记录到 `volfile-checksum`
（`xlators/protocol/client/src/client-handshake.c:778`），用于判断是否需要重新获取。

## 变更如何生效

`glusterd` 修改配置后重新生成 volfile 并推送给相关进程，进程侧由
`glusterfs_volfile_reconfigure`（`libglusterfs/src/graph.c:961`）处理，它的返回值区分
两条路径：

| 返回值 | 条件 | 后续动作 |
| --- | --- | --- |
| `0` | 新旧图的拓扑相同，只有选项不同 | 就地重配置，不重建节点 |
| `1` | 拓扑发生变化 | 由调用方重建整张图并重新初始化所有 xlator |
| 负值 | 内部错误 | 保持原图 |

判断拓扑是否相同由 `is_graph_topology_equal`（`libglusterfs/src/graph.c:916`）完成。
就地重配置最终落到 `glusterfs_graph_reconfigure`
（`libglusterfs/src/graph.c:1143`），逐节点调用其 `reconfigure` 方法。图被替换后，旧图
的资源由 `glusterfs_graph_destroy_residual`（`libglusterfs/src/graph.c:1186`）清理。

这解释了一个常见现象：修改一个 xlator 的选项通常不中断服务，而增删 brick 或改变卷类型
会导致整张图重建。

## 阅读一份 volfile 的方法

1. 先找到叶子节点：数据最终落到 `storage/posix`，其 `option directory` 是 brick 的真实
   路径；
2. 由下往上逐层识别：紧邻存储之上通常是特性层（锁、配额、索引等），再往上按需是性能层；
3. 区分两类顶层：服务端 volfile 的顶层是 `protocol/server`，客户端 volfile 的顶层一般
   是挂载层或统计层，其下方才是性能层与集群层；
4. 客户端与服务端通过 `protocol/client` 与 `protocol/server` 的成对配置对接，前者用
   `remote-host`、`remote-port`、`remote-subvolume` 指向后者。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 语法定义与解析器 | `libglusterfs/src/graph.y`、`libglusterfs/src/graph.l` |
| 图构建、准备、激活 | `libglusterfs/src/graph.c:590`、`:805` |
| 变更处理 | `libglusterfs/src/graph.c:961`、`:1143` |
| volfile 生成 | `xlators/mgmt/glusterd/src/glusterd-volgen.c:1070` |
| 示例 volfile | `extras/thin-arbiter/thin-arbiter.vol` |

## 相关文档

- [xlator 框架](xlator-framework.md)
- [RPC 与传输](rpc-and-transport.md)
- [架构总览](../overview/architecture.md)
