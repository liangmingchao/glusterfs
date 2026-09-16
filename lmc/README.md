# GlusterFS 代码知识库

本目录（`lmc/`）是 GlusterFS 代码库的结构化工程知识库，用来回答三类问题：某个能力在
哪里实现、为什么这样设计、改动时会影响什么。正文以中文撰写，代码标识符、路径、配置
项、命令保留英文原文。

`lmc/` 是本地新增目录，不属于 GlusterFS 上游代码，不参与构建，也不会被 `make`、
`configure` 或测试框架读取。

## 基线版本

知识库描述的代码以本仓库的以下提交为基线，全库统一。

| 项目 | 值 |
| --- | --- |
| 仓库 | GlusterFS（`origin` = `https://github.com/liangmingchao/glusterfs.git`） |
| 分支 | `devel` |
| 提交 | `a482a8578a6d8ff2a9f5ee1c63696af5dfdfd88a`（2026-09-11） |

文档中所有 `路径:行号` 形式的代码引用都指向该提交。基线推进时按
[更新规则](meta/update-rules.md) 同步校正引用，其他文档不重复声明基线。

## 知识库地图

| 文档 | 主题 |
| --- | --- |
| [架构总览](overview/architecture.md) | 系统组成、进程模型、xlator 栈、一次 I/O 的完整路径 |
| [代码地图](overview/code-map.md) | 顶层目录与 `xlators/` 分类的职责划分、关键文件 |
| [术语表](overview/glossary.md) | 全库统一使用的术语定义 |
| [构建与测试](overview/build-and-test.md) | 构建方式、测试框架与常用命令 |
| [xlator 框架](core/xlator-framework.md) | xlator 的导出接口、生命周期、加载与组装 |
| [调用栈与 FOP 模型](core/call-stack-and-fop.md) | FOP 集合、call_frame/call_stack、STACK_WIND/UNWIND、同步接口 |
| [volfile 与图构建](core/volfile-and-graph.md) | volfile 语法、graph 构建流程、glusterd 侧生成逻辑 |
| [RPC 与传输](core/rpc-and-transport.md) | RPC 框架、XDR、连接建立与握手 |
| [模块索引](modules/README.md) | 按模块（xlator、子系统）组织的文档 |
| [特性索引](features/README.md) | 按特性组织的文档 |
| [更新规则](meta/update-rules.md) | 知识库的强制写作与维护规范 |

## 推荐阅读路径

按目标选择入口，避免从头顺序读。

| 目标 | 阅读顺序 |
| --- | --- |
| 建立整体认知 | 架构总览 → 代码地图 → xlator 框架 |
| 读懂一次文件操作 | 调用栈与 FOP 模型 → 架构总览的「一次写请求的完整路径」→ RPC 与传输 |
| 定位某个功能代码 | 代码地图 → 模块索引 / 特性索引 |
| 搭建开发与验证环境 | 构建与测试 |
| 新增或修改知识库文档 | 更新规则 → 对应模板 |

## 覆盖范围

已收录：整体架构、代码地图、术语、构建与测试、xlator 框架、调用栈与 FOP 模型、
volfile 与图构建、RPC 与传输；模块文档 `dict`、`inode`、`fd`、`snapview-server`、
`snapview-client`；特性文档 `快照`。

按模块与特性的深入分析在 [模块索引](modules/README.md) 与
[特性索引](features/README.md) 中维护，两者是该层次内容唯一的登记入口。

## 维护须知

新增或修改任何文档前先读 [更新规则](meta/update-rules.md)。最容易被忽略的一条是：
文档只呈现终态，不保留修订记录、旧结论和过程性描述。
