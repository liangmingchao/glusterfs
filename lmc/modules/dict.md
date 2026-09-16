# dict

`dict_t` 是 GlusterFS 内部通用的键值容器，用于承载选项集合、随 FOP 传递的 xdata、管理
命令的参数与应答、以及持久化元数据。它的值带有类型标签，支持引用计数与跨网络序列化，
是全库出现频率最高的数据结构之一。

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

dict 出现在几乎所有跨层接口上：

| 场景 | 说明 |
| --- | --- |
| xlator 选项 | volfile 中的 `option` 解析后存入 `xlator_t.options`，由 `GF_OPTION_INIT` 读取 |
| FOP 附加数据 | 每个 FOP 的 `xdata` 参数，用于传递不属于标准参数的扩展信息 |
| 协议载荷 | 跨节点传输时序列化为字节流，见「序列化」一节 |
| 管理配置 | `glusterd` 的卷配置、`store` 转换、快照元数据 |
| 运维观测 | statedump 与日志转储 |

## 核心概念

**键与值。** 键是 C 字符串，值是 `data_t`，由「字节串 + 类型标签 + 长度」组成。类型
标签取自 `gf_dict_data_type_t`（`libglusterfs/src/glusterfs/glusterfs-fops.h:227`），
包含字符串、各类整数、双精度、指针、GFID、`iatt`、元数据 `iatt` 等取值。

**值的所有权。** 同一个键可以绑定不同生命周期的值对象：

| 取值方式 | 行为 | 典型用途 |
| --- | --- | --- |
| 拷贝字符串（`dict_set_strn`） | dict 复制一份，原指针可立即释放 | 一般字符串配置 |
| 接管指针（`dict_set_dynstrn`） | dict 直接持有，销毁时释放 | 已经分配好、不再复用的字符串 |
| 静态引用（`dict_set_static_ptr`） | 只存指针，销毁时不释放 | 指向常量或由调用方管理的缓冲区 |

所有权由 `data_t.is_static` 记录，销毁时据它决定是否释放底层字节
（`libglusterfs/src/dict.c:276`）。

**引用计数。** `dict_t` 与 `data_t` 各自带原子引用计数，见
`dict_ref`/`dict_unref`（`libglusterfs/src/dict.c:600`、`:583`）与
`data_ref`/`data_unref`（`libglusterfs/src/dict.c:630`、`:613`）。`dict_copy_with_ref`
（`libglusterfs/src/dict.c:1292`）在拷贝时共享值对象而不复制内容。

**编译期键长。** 主要接口都有一组带 `n` 后缀的版本显式传入键长，`_sizen` 形式的宏用
`SLEN` 在编译期求出键长，避免运行时 `strlen`（`libglusterfs/src/glusterfs/dict.h:24`）。

## 数据结构

| 结构体 | 位置 | 作用 |
| --- | --- | --- |
| `struct _data` | `libglusterfs/src/glusterfs/dict.h:96` | 值对象：字节内容、引用计数、类型标签、长度、是否静态 |
| `struct _data_pair` | `libglusterfs/src/glusterfs/dict.h:104` | 链表节点，`key` 是跟在结构体之后的柔性数组 |
| `struct _dict` | `libglusterfs/src/glusterfs/dict.h:110` | 容器本身：计数、长度合计、锁与链头 |

`struct _dict` 中有四个字段决定它的行为：

| 字段 | 含义 |
| --- | --- |
| `members_list` | 键值对链表的表头，新键插入表头 |
| `count` / `max_count` | 当前键数量与历史峰值 |
| `totkvlen` | 所有「键长 + 1 + 值长」之和，使序列化长度可在 O(1) 算出 |
| `extra_stdfree` | 由外部托管、随 dict 一起释放的缓冲区指针 |

链表是理解 dict 性能特征的关键：**它没有哈希索引**，查找由 `dict_lookup_common`
（`libglusterfs/src/dict.c:322`）线性扫描完成。dict 的定位是承载少量键值对的传递型
容器，不适合当作大表使用。

`extra_stdfree` 是协议层的惯用手法：解码应答后把 XDR 分配的缓冲区指针挂到 dict 上，
由 dict 的销毁顺带释放，避免单独管理生命周期，例如
`glusterfsd/src/glusterfsd-mgmt.c:2336`。

## 工作流程

### 创建与销毁

`dict_new`（`libglusterfs/src/dict.c:83`）从 `ctx->dict_pool` 内存池取内存并初始化锁
（`get_new_dict_full`，`libglusterfs/src/dict.c:68`），引用计数置 1。`dict_destroy`
（`libglusterfs/src/dict.c:541`）释放全部键值对与 `extra_stdfree`，并把本次 dict 的
键数量统计累加到全局计数器。

### 插入、替换与删除

`dict_setn`（`libglusterfs/src/dict.c:397`）与 `dict_addn`
（`libglusterfs/src/dict.c:419`）共用 `dict_set_lk`
（`libglusterfs/src/dict.c:360`），行为差别只在于是否做重复键检查：

| 接口 | 同键已存在时的行为 |
| --- | --- |
| `dict_setn` / `dict_set` | 替换值并修正 `totkvlen`，不新增链表节点 |
| `dict_addn` / `dict_add` | 不做重复检查，直接插入新节点，允许同键并存 |

删除由 `dict_deln`（`libglusterfs/src/dict.c:482`）完成，只删除第一个匹配节点，删除成功
返回 `_gf_true`。

### 遍历

三种遍历方式，按「是否需要按条件筛选」与「是否需要加锁」选择：

| 接口 | 特点 |
| --- | --- |
| `dict_foreach_inline`（`libglusterfs/src/glusterfs/dict.h:86`） | 宏，直接遍历链表，不加锁，调用方负责并发安全 |
| `dict_foreach`（`libglusterfs/src/dict.c:1137`） | 加锁遍历，回调返回非零可提前终止 |
| `dict_foreach_match`（`libglusterfs/src/dict.c:1155`） | 先按匹配函数筛选，再对命中的项执行动作 |

### 序列化

跨节点传输时 dict 被压成自描述的字节流，长度计算与写入分别由
`dict_serialized_length_lk`（`libglusterfs/src/dict.c:2747`）与
`dict_serialize_lk`（`libglusterfs/src/dict.c:2769`）完成，布局为：

| 顺序 | 内容 |
| --- | --- |
| 1 | 4 字节大端整数：键值对数量 |
| 2 | 每对键值重复：4 字节大端键长、4 字节大端值长、键字节（含结尾 `\0`）、值字节 |

相关常量 `DICT_HDR_LEN`、`DICT_DATA_HDR_KEY_LEN`、`DICT_DATA_HDR_VAL_LEN` 定义在
`libglusterfs/src/glusterfs/dict.h:92`。字节流中只记录键与值的原始字节，**不记录类型
标签**，接收端统一按字节串还原。

发送侧入口为 `dict_allocate_and_serialize`（`libglusterfs/src/dict.c:2974`），接收侧为
`dict_unserialize`（`libglusterfs/src/dict.c:2837`）。协议代码通常直接使用封装宏
`GF_PROTOCOL_DICT_SERIALIZE` 与 `GF_PROTOCOL_DICT_UNSERIALIZE`
（`libglusterfs/src/glusterfs/dict.h:61`）；只关心部分键时使用
`dict_unserialize_specific_keys`（`libglusterfs/src/dict.c:3241`）。

## 关键分支与边界条件

| 条件 | 行为 | 位置 |
| --- | --- | --- |
| 值的长度超过 `DICT_KEY_VALUE_MAX_SIZE`（1 MiB） | 返回 `-EINVAL`，不插入 | `libglusterfs/src/glusterfs/dict.h:88`、`libglusterfs/src/dict.c:2477` |
| 传入的 dict 或 key 为 NULL | 记录日志并返回失败 | 各接口入口 |
| 删除时存在同键的多份内容 | 只删除第一个匹配节点 | `libglusterfs/src/dict.c:482` |
| 序列化时链表节点数少于 `count` | 记录错误并中止，避免越界读写 | `libglusterfs/src/dict.c:2769` |

## 并发与锁

每把 dict 自带一把 `gf_lock_t`，所有修改与遍历操作都在锁内完成，因此多个线程可以安全地
对同一 dict 做插入、查询与删除。两条必须遵守的约束：

- `dict_foreach` 在整个遍历期间持锁，回调函数内**不能**再对同一 dict 加锁，否则死锁；
  需要在遍历中修改时使用 `dict_foreach_inline`，或先收集键再处理；
- 以 `_lk` 结尾的内部函数（`dict_set_lk`、`dict_serialize_lk`、
  `dict_serialized_length_lk`）要求调用方已经持锁，它们自身不加锁。

`data_t` 的引用计数是原子操作，增减不需要持有 dict 锁。

## 与其他模块的交互

| 交互对象 | 形式 |
| --- | --- |
| xlator 选项 | `xlator_t.options` 是一个 dict，由 `GF_OPTION_INIT`、`GF_OPTION_RECONF` 读取 |
| FOP 参数 | 每个 FOP 的 `xdata` 参数即 `dict_t *` |
| RPC 编解码 | 通过上述序列化接口与 `rpc/xdr/src/glusterfs3.h` 中的编解码逻辑对接 |
| 持久化 | `libglusterfs/src/store.c` 负责 dict 与磁盘文件之间的转换 |
| 内存池 | 从 `ctx->dict_pool` 分配，池由全局上下文管理 |

## 可观测性

- `dict_dump_to_log`（`libglusterfs/src/dict.c:3116`）把内容写入日志；
- `dict_dump_to_statedump`（`libglusterfs/src/dict.c:3150`）把内容写入 statedump 段；
- `dict_destroy` 更新全局统计 `ctx->stats.max_dict_pairs`、`total_pairs_used`、
  `total_dicts_used`（字段定义见 `libglusterfs/src/glusterfs/glusterfs.h:748`），
  statedump 的 dict 段即来自这些计数器。

## 代码位置

| 内容 | 位置 |
| --- | --- |
| 类型与接口声明 | `libglusterfs/src/glusterfs/dict.h` |
| 实现 | `libglusterfs/src/dict.c` |
| 值类型枚举 | `libglusterfs/src/glusterfs/glusterfs-fops.h:227` |
| 协议侧编解码 | `rpc/xdr/src/glusterfs3.h` |

## 相关文档

- [inode 与 inode 表](inode.md)
- [fd](fd.md)
- [xlator 框架](../core/xlator-framework.md#选项)
- [RPC 与传输](../core/rpc-and-transport.md)
- [术语表](../overview/glossary.md)
