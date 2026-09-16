# 构建与测试

本文说明如何从源码构建 GlusterFS、各构件安装到哪里，以及如何运行仓库自带的两类测试。
内容面向「需要编译改动并验证」的场景。

## 依赖与环境

构建使用 autotools。从 git 检出后需要先生成 `configure`，`configure` 阶段会检查
FUSE、readline、georeplication、Linux AIO、块设备支持等组件，未满足的可选组件会导致
对应功能不参与构建，检查结果在配置摘要中逐项打印（`INSTALL`）。

## 构建步骤

```bash
./autogen.sh
./configure
make
make install
```

其中 `./autogen.sh` 仅在从 git 检出时需要执行，用于生成 `configure`
（`INSTALL`）。构建产物由顶层 `Makefile.am` 的 `SUBDIRS` 顺序决定：核心库与 RPC 先于
使用它们的模块，translator 在 `xlators` 子目录内构建
（`Makefile.am:13`）。

## 安装布局

| 内容 | 位置 |
| --- | --- |
| 主可执行文件 | `<sbindir>/glusterfsd` |
| `glusterfs`、`glusterd` | 指向 `glusterfsd` 的符号链接（`glusterfsd/src/Makefile.am:51`、`glusterfsd/src/Makefile.am:54`） |
| xlator 共享库 | `$(libdir)/glusterfs/$(PACKAGE_VERSION)/xlator`（`libglusterfs/src/Makefile.am:9`） |
| 配置文件 | `$(sysconfdir)/glusterfs`（`glusterfsd/src/Makefile.am:23`） |
| 运行期数据 | `$(localstatedir)`，其中运行目录见 `libglusterfs/src/glusterfs/glusterfs.h:281`，管理数据目录见 `libglusterfs/src/glusterfs/glusterfs.h:284` |

进程运行期的角色由可执行文件的名字决定，因此验证改动时必须确认调用的是期望的角色，见
[架构总览](architecture.md#单二进制多角色)。

## 功能测试

`tests/` 下是回归测试，主体为 `.t` 脚本，由 `run-tests.sh` 批量执行。测试默认以 root
运行，并且会清理本机所有 gluster 进程，只应在专用测试机上执行（`README.md`）。

### 测试脚本结构

每个 `.t` 文件是一个 bash 脚本，开头引入公共框架，末尾清理环境：

```bash
. $(dirname $0)/../include.rc
. $(dirname $0)/../volume.rc

cleanup;

TEST glusterd
TEST $CLI volume create $V0 replica 3 $H0:$B0/${V0}{1,2,3,4,5,6,7,8,9};
EXPECT 'Created' volinfo_field $V0 'Status';
cleanup;
```

关键约定：

| 元素 | 作用 | 定义位置 |
| --- | --- | --- |
| `TEST` | 执行命令并检查退出码，`LINENO` 用于定位失败行 | `tests/include.rc:1025`（别名到 `_TEST`） |
| `EXPECT` | 比较命令输出与期望值 | `tests/include.rc:1020`（别名到 `_EXPECT`，实现见 `tests/include.rc:266`） |
| `cleanup` | 结束本次测试并回收进程、挂载点、临时目录 | `tests/include.rc` |
| `$CLI`、`$GFS`、`$V0`、`$B0`、`$H0` | CLI 路径、glusterfs 路径、卷名、brick 根目录、主机名等公共变量 | `tests/include.rc` 顶部 |
| `tests/*.rc` | 按主题提供的辅助函数，例如 `volume.rc`、`afr.rc`、`dht.rc` | `tests/` 根目录 |

测试约定不要用 `mount -t glusterfs`，而应使用 `$GFS -s <host> --volfile-id <vol> <mnt>`，
原因见 `tests/README.md`。

### 执行方式

```bash
./run-tests.sh                      # 全量
./run-tests.sh basic/               # 按路径过滤
bash tests/basic/rpc-coverage.t     # 单个脚本，需在脚本内自行准备环境
prove tests/basic/...               # 使用 prove 驱动
```

`run-tests.sh` 的常用选项（`run-tests.sh:545`）：

| 选项 | 含义 |
| --- | --- |
| `-H` | 只跑相对 HEAD 有改动的测试 |
| `-l` | 只列出将被执行的测试，不实际运行 |
| `-c` | 遇到失败继续执行 |
| `-R` | 失败不重试 |
| `-t TIMEOUT` | 设置单个测试超时 |
| `-p` | 不保留上一轮的日志归档 |

## 单元测试

`libglusterfs` 下带一组基于 cmocka 的单元测试，测试代码位于
`libglusterfs/src/unittest/`（例如 `mem_pool_unittest.c`），桩函数在
`libglusterfs/src/unittest/global_mock.c` 与 `libglusterfs/src/unittest/log_mock.c`。
该组测试需在配置阶段显式开启（`configure.ac:1681` 的 `--enable-cmocka`）。

## 相关文档

- [架构总览](architecture.md)
- [代码地图](code-map.md)
