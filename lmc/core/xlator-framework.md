# xlator 框架

xlator 是 GlusterFS 的功能单元：一个共享库，导出唯一符号 `xlator_api`，声明自己的
初始化方法、文件操作能力表与选项集合。进程启动时按 volfile 描述加载若干 xlator 并组装
成树；运行期所有文件操作都在这些 xlator 之间逐层传递。本文说明 xlator 的对外接口、
加载过程与生命周期，请求如何在它们之间传递见
[调用栈与 FOP 模型](call-stack-and-fop.md)。

## 目录

- 对外接口：xlator_api
- 运行时实例：xlator_t
- 能力表
- 加载过程
- 生命周期
- 选项
- 能力等级与版本
- 新增一个 xlator 需要提供的内容
- 代码位置

## 对外接口：xlator_api

加载器只认一个符号名 `xlator_api`，其类型为
`xlator_api_t`（`libglusterfs/src/glusterfs/xlator.h:854`）：

```c
typedef struct {
    uint32_t op_version[GF_MAX_RELEASES];
    char *identifier;
    volume_option_t *options;
    gf_category_t category;
    int32_t (*init)(xlator_t *this);
    void (*fini)(xlator_t *this);
    int32_t (*reconfigure)(xlator_t *this, dict_t *options);
    int32_t (*mem_acct_init)(xlator_t *this);
    int32_t (*dump_metrics)(xlator_t *this, int fd);
    event_notify_fn_t notify;
    struct xlator_fops *fops;
    struct xlator_cbks *cbks;
    struct xlator_dumpops *dumpops;
    struct xlator_fops *pass_through_fops;
} xlator_api_t;
```

各字段的强制程度：

| 字段 | 是否必需 | 说明 |
| --- | --- | --- |
| `init` | 必需 | 缺失时加载直接失败，见 `xlator_dynload_apis`（`libglusterfs/src/xlator.c:262`） |
| `fops` | 必需 | 缺失时加载失败；未实现的具体操作由框架补默认实现 |
| `cbks`、`fini`、`reconfigure`、`notify`、`dumpops` | 可选 | 缺失时使用默认行为 |
| `options` | 可选 | 该 xlator 接受的配置项集合 |
| `op_version` | 可选 | 参与卷生成时的版本判定 |
| `category` | 可选 | 能力等级声明 |

该结构有一处修改约束：新增成员必须加在 `GD2MARKER` 注释之前或结构体末尾，因为外部组件
按固定布局读取该结构（`libglusterfs/src/glusterfs/xlator.h:871`）。

一个最小示例是 `posix` 的注册块
（`xlators/storage/posix/src/posix.c:87`）：

```c
xlator_api_t xlator_api = {
    .init = posix_init,
    .fini = posix_fini,
    .notify = posix_notify,
    .reconfigure = posix_reconfigure,
    .op_version = {1},
    .dumpops = &dumpops,
    .fops = &fops,
    .cbks = &cbks,
    .options = posix_options,
    .identifier = "posix",
    .category = GF_MAINTAINED,
};
```

## 运行时实例：xlator_t

`xlator_api_t` 是静态的能力声明，进程内每个被实例化的节点对应一个 `xlator_t`
（`libglusterfs/src/glusterfs/xlator.h:750`）。理解其中几组字段即可读懂大部分代码：

| 字段组 | 含义 |
| --- | --- |
| `name`、`type`、`instance_name` | 实例名、模块类型（如 `storage/posix`）、多实例场景下的区分名 |
| `parents`、`children` | 树中的上下级关系，决定 wind 的目标 |
| `options` | 本次实例化生效的配置 |
| `dlhandle`、`fops`、`cbks`、`dumpops` | 加载后的句柄与能力表指针，来自 `xlator_api` |
| `init`、`fini`、`reconfigure`、`notify` | 从 `xlator_api` 复制过来的方法指针 |
| `private` | 该 xlator 自己的状态，由 `init` 分配、`fini` 释放 |
| `graph`、`ctx`、`itable` | 所属图、全局上下文、inode 表 |
| `stats[GF_FOP_MAXVALUE]` | 每个 FOP 的调用计数与延迟统计 |
| `pass_through`、`pass_through_fops` | 直通模式的开关与其能力表 |

## 能力表

三个结构体描述一个 xlator 能做什么：

| 结构体 | 位置 | 内容 |
| --- | --- | --- |
| `struct xlator_fops` | `libglusterfs/src/glusterfs/xlator.h:546` | 文件操作实现，如 `lookup`、`readv`、`writev`、`setxattr` |
| `struct xlator_cbks` | `libglusterfs/src/glusterfs/xlator.h:692` | 生命周期回调，如 `forget`、`release`、`releasedir` |
| `struct xlator_dumpops` | `libglusterfs/src/glusterfs/xlator.h:731` | 状态转储入口，供 statedump 使用 |

一个 xlator 不必实现全部 FOP。加载后由 `fill_defaults`
（`libglusterfs/src/xlator.c:81`）为每个未实现的 FOP 填入 `default_<fop>`，默认实现的
行为是把请求原样转发给第一个子节点，并在回调中直接 unwind。

这些默认实现不是手写的，而是在构建期由 `libglusterfs/src/defaults-tmpl.c` 经
`libglusterfs/src/gen-defaults.py` 生成（`libglusterfs/src/Makefile.am:136`）。
另有 `libglusterfs/src/default-args.c`，它提供的是一组 `args_<fop>_store` 函数，用于
暂存 FOP 参数以便后续恢复，与默认实现是两回事。

## 加载过程

volfile 中每个节点的 `type` 最终变成一次 `dlopen`：

| 步骤 | 位置 |
| --- | --- |
| 拼接库路径 `XLATORDIR/<type>.so` 并 `dlopen` | `libglusterfs/src/xlator.c:374`（`xlator_dynload`） |
| 取符号 `xlator_api` 并复制方法与能力表 | `libglusterfs/src/xlator.c:262`（`xlator_dynload_apis`） |
| 填充未实现的 FOP 与回调 | `libglusterfs/src/xlator.c:81`（`fill_defaults`） |

`XLATORDIR` 的取值是 `$(libdir)/glusterfs/$(PACKAGE_VERSION)/xlator`
（`libglusterfs/src/Makefile.am:9`）。因此运行新编译的 xlator 必须用到对应安装目录，
直接运行源码树中的二进制不会加载源码树中的 `.so`。

除按类型加载外，还有一条只读选项表的路径：`xlator_volopt_dynload`
（`libglusterfs/src/xlator.c:181`）把模块加载进来只为读取 `options`，供管理端校验配置
使用。该函数对 `rpc-transport` 开头的类型会改到父目录查找。

## 生命周期

一个 xlator 实例经历以下阶段，任一阶段失败都会导致所属图构建失败：

| 阶段 | 触发时机 | 实现 |
| --- | --- | --- |
| 实例化 | 解析 volfile 时按节点创建 | `libglusterfs/src/graph.y` |
| 初始化 | 图激活前，自底向上依次调用 | `libglusterfs/src/xlator.c:624`（`xlator_init`） |
| 重配置 | 卷配置变化但图结构不变时 | `reconfigure`，由 `libglusterfs/src/graph.c:1143`（`glusterfs_graph_reconfigure`）驱动 |
| 事件通知 | 上下级状态变化时 | `notify`，由 `libglusterfs/src/xlator.c:706`（`xlator_notify`）分发 |
| 销毁 | 图退出或被替换 | `fini` |

事件通知与文件操作是两条独立通道：`notify` 传递的是「子卷就绪」「连接断开」「图切换」
等状态变化，不携带文件数据。事件编号定义在
`libglusterfs/src/glusterfs/events.h`。

## 选项

一个 xlator 接受的配置项在 `xlator_api_t.options` 中以 `volume_option_t`
（`libglusterfs/src/glusterfs/options.h:160`）数组声明，包含键名、类型、默认值、
取值范围与中文/英文说明。配置项在 volfile 中写作 `option <key> <value>`。

读取选项有两种宏，分别对应首次初始化和重配置：

| 宏 | 位置 | 用途 |
| --- | --- | --- |
| `GF_OPTION_INIT` | `libglusterfs/src/glusterfs/options.h:269` | 在 `init` 中读取并校验选项，失败跳到指定的清理标签 |
| `GF_OPTION_RECONF` | `libglusterfs/src/glusterfs/options.h:330` | 在 `reconfigure` 中读取，支持「未提供则保持原值」 |

## 能力等级与版本

`category` 字段声明 xlator 的成熟度，取值范围见 `gf_category_t`
（`libglusterfs/src/glusterfs/glusterfs.h:434`）：`GF_EXPERIMENTAL`、`GF_TECH_PREVIEW`、
`GF_MAINTAINED`、`GF_DEPRECATED`、`GF_OBSOLETE`、`GF_DOCUMENT_PURPOSE`。

`op_version` 与 `GF_MAX_RELEASES`（`libglusterfs/src/glusterfs/options.h:85`）配合，用于
在卷生成阶段判断某个 xlator 是否应进入图，从而使新旧版本的节点可以在同一个集群中共存。

## 新增一个 xlator 需要提供的内容

1. 一个 `<name>.c`，其中定义 `fops`、`cbks`、`dumpops` 与唯一的 `xlator_api`；
2. 非空的 `init`，用于分配 `private` 并读取选项；
3. 需要接管的 FOP 的完整实现，未实现的部分交由框架的默认实现处理；
4. 接受的配置项表 `volume_options`；
5. 构建脚本中的模块声明，使产物安装到 `XLATORDIR`。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 对外接口定义 | `libglusterfs/src/glusterfs/xlator.h:854` |
| 运行时实例定义 | `libglusterfs/src/glusterfs/xlator.h:750` |
| 能力表定义 | `libglusterfs/src/glusterfs/xlator.h:546`、`:692`、`:731` |
| 加载与默认填充 | `libglusterfs/src/xlator.c` |
| 默认实现模板 | `libglusterfs/src/defaults-tmpl.c` |
| 参考实现 | `xlators/storage/posix/src/posix.c:87` |

## 相关文档

- [架构总览](../overview/architecture.md)
- [调用栈与 FOP 模型](call-stack-and-fop.md)
- [volfile 与图构建](volfile-and-graph.md)
- [术语表](../overview/glossary.md)
