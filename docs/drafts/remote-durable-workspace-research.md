# 从 TigerFS 到 Remote Durable Workspace

## PostgreSQL 作为云端 Agent 工作区底座的调研报告

- 调研日期：2026-10-01
- 研究对象：当前仓库中的 TigerFS、分享对话中的 PGFS 设计讨论，以及相关 PostgreSQL / FUSE / CSI 资料
- 当前仓库状态：`main`，`HEAD 96d41a9`，仓库远端为 `https://github.com/wubuku/tigerfs.git`
- 报告性质：技术架构调研与实施建议，不是对分享对话的逐字整理，也不是对 TimescaleDB 商业许可的法律意见

## 摘要

这次讨论表面上围绕“能否把 PostgreSQL 当作文件系统”展开，实际逐步收敛到了一个更有价值、也更可实施的产品命题：

> 不要把 Agent 改造成懂云存储的 Agent；应把云端持久化层做成 Agent 认为自己正在使用的 `/workspace`。

这一区分决定了项目边界：目标不是重新实现 ext4、xfs 或一个完整的 POSIX 内核文件系统，而是为代码 Agent、Shell、编辑器、Git 和包管理器提供足够可靠的工作区文件语义，并把工作区的持久化、版本、回滚、隔离和共享下沉到云端基础设施。

结合分享对话与 TigerFS 当前代码，结论如下：

1. **方向可行，但当前 TigerFS 不能直接等价为云端多租户 PGFS。** TigerFS 已经证明了 PostgreSQL-backed transactional workspace 的可用性，但它当前的主模型是“数据库映射 + 合成文件工作区”，而不是通用的二进制、inode/dentry 文件系统。
2. **TigerFS 最值得复用的是语义和工程经验，而不是直接把现有表结构当成最终云端存储模型。** 共享核心、原子重命名、父目录关系、历史、保存点、撤销、缓存和压力测试都是重要资产；但多租户控制面、远程存储服务、幂等 RPC、租户级 RLS、对象/分块内容层仍需单独设计。
3. **TimescaleDB 的必要性必须区分“当前 TigerFS”与“未来 PGFS”。** 当前仓库在启用 history 时检查 TimescaleDB，并在历史表和操作日志上使用 `tsdb.hypertable`、`tsdb.segmentby`、`tsdb.orderby`；这些 DDL 在 Community 模式下启用压缩/columnstore，而在 Apache 模式下仍可建表但不会得到同等压缩状态。官方 edition matrix 将 hypertable/chunk 基础能力列入 Apache 2 Edition，将 Hypercore/columnstore、continuous aggregates、retention/jobs 等列入 Community Edition。因此，“新 PGFS 可以不依赖 TSL”是合理的架构目标，但不能反推“当前 TigerFS history 已经是 Apache-only”。
4. **第一阶段不应先做完整分布式文件系统。** 最有价值的 PoC 是一个只覆盖 Agent 工作负载的 WorkspaceFS：`lookup/readdir/open/read/write/mkdir/rename/unlink/stat`，使用 PostgreSQL 持久化，在真实 Codex、Claude Code、Git、Node/Python 项目上验证“无需修改 Agent 代码”是否成立。
5. **真正的安全边界应在 Storage Service 和数据库授权层，而不是 FUSE。** 在 SaaS 场景中，不应把 `/dev/fuse`、`SYS_ADMIN` 或 mount 权限直接交给不可信 Agent 容器；客户端或节点侧适配器应尽量薄，租户身份应由受信控制面注入，并在每个数据库事务内通过 `SET LOCAL` 设置上下文。
6. **当前 TigerFS 可以在 macOS 上运行，不是 Linux-only。** macOS 使用系统自带的 `mount_nfs` 加载 TigerFS 的进程内 NFS 服务，Linux 才使用 FUSE；共享的数据库和文件系统核心不依赖 Linux 特有的文件系统抽象。但两平台的缓存、锁、写入分块和故障语义不同，不能把 macOS NFS 路径当成 Linux FUSE 的完全等价实现。

最终建议将产品抽象命名为 `WorkspaceFS` 或 `Remote Durable Workspace`，而不是把产品本身命名为 `PostgreSQLFS`。PostgreSQL 是第一种后端，文件系统语义、租户隔离、版本与恢复才是产品边界。

## 1. 研究范围与证据方法

### 1.1 分享对话的结构

分享页实际包含五条消息，内容可归纳为三个阶段：

- **架构论证**：从 PostgreSQL-backed filesystem、TigerFS、Tarbox 和传统 PostgreSQL-FUSE 实现出发，提出“Remote Filesystem Compatibility Layer”的概念。
- **许可追问**：进一步讨论 TimescaleDB 的 Apache 2.0 / Timescale License 分层，以及这对商业 SaaS 的影响。
- **技术设计**：把前面的判断工程化为 PGFS Schema、租户隔离、FUSE/CSI 部署、缓存、一致性、版本和 Agent 兼容性测试矩阵。

本报告保留这些讨论的思想脉络，但重新按“问题—事实—架构—实施—风险”的顺序组织，避免把对话中的假设、建议和已实现能力混在一起。

### 1.2 核查优先级

本报告采用以下证据优先级：

1. 当前仓库代码、测试、ADR 和文档；
2. PostgreSQL、FUSE、CSI、TimescaleDB 的官方资料；
3. 分享对话中的设计建议；
4. 项目 Star、Fork、第三方采用情况等易变指标只作背景，不作为技术结论。

需要特别避免把分享对话中“约多少 Stars”“最新版本是什么”等随时间变化的信息当成架构证据。当前仓库的代码和测试才是判断 TigerFS 实际能力的主要依据。

## 2. 问题重述：真正要云化的是工作区，不是整个操作系统

传统本地 Agent 的隐含运行模型通常是：

```text
Agent
  ├── shell / bash
  ├── read / write / edit
  ├── grep / find / rg
  ├── git
  └── code execution
           │
           ▼
     /workspace
           │
           ▼
     local filesystem
```

一旦 Agent 变成 SaaS，多次运行可能发生在不同机器、不同容器或不同时间点。若 `/workspace` 仍绑定本地盘，就需要额外解决：

- 会话结束后的持久化；
- 任务迁移与容器重建；
- 多 Agent / 人类协作；
- 并发写入和冲突检测；
- 版本、快照、恢复；
- 租户隔离、审计和配额；
- 节点扩缩容及数据库连接管理。

直接给 Agent 一个 S3 API 或自定义 Remote File API，会把同步、flush、冲突和重试责任推回 Agent。更合理的结构是：

```text
Existing Agent
      │ 只看到 /workspace
      ▼
Filesystem Compatibility Layer
      │ FUSE / NFS / CSI / HTTP 等适配器
      ▼
Remote Durable Workspace Service
      │ 事务、鉴权、缓存、版本、配额
      ▼
PostgreSQL + 可选 Object Storage
```

这里的核心创新不是“数据库可以存文件”，而是把本地文件系统依赖从 Agent runtime 中抽出来，变成一个可独立扩展的云基础设施层。

## 3. TigerFS 当前实现的事实底图

### 3.1 当前产品定位

仓库 README 把 TigerFS 定义为“由 PostgreSQL 支撑的版本化文件系统”和“PostgreSQL 的文件系统接口”，并明确提供两种模式：

- **File-first**：通过 Markdown / Plain Text 等合成工作区，把文件和目录写入 PostgreSQL；
- **Data-first**：把已有 PostgreSQL 的 schema、table、row、column 映射成目录、文件和路径能力。

README 还将 history、log、savepoint、undo、SQL pushdown 和 Agent skills 放在同一产品表面上。这个组合非常适合“让 Agent 通过熟悉的文件工具探索和操作数据库”，但它与“给任意程序提供一块通用远程 POSIX 盘”仍有明显差别。

证据：`README.md:3-28`、`docs/spec.md:41-57`。

### 3.1.1 参考实现的正确定位

分享页还比较了 TigerFS、Tarbox 和较早的 PostgreSQL-FUSE 项目。这个比较有价值的地方不在于 Star 数，而在于它们覆盖了不同的设计轴：

- **TigerFS**：更接近 Agent-friendly 的数据库工作区，重点是 File-first / Data-first、事务、history、savepoint、undo、SQL pushdown 和跨平台适配。
- **Tarbox**：分享页将其作为 PostgreSQL-backed POSIX / cloud-native filesystem 的参考方向，强调内容寻址、多租户、layer/COW、REST/gRPC 和 CSI；它更适合启发通用文件系统与云部署问题。
- **传统 PostgreSQL-FUSE 项目**：更接近把数据库结构映射成目录的浏览器式文件系统，能证明“数据库对象可以被文件接口访问”，但不能直接证明 Agent 工作区、版本和 SaaS 隔离已经解决。

因此，本报告不把这些项目的关注度或 Alpha/early 状态当作生产成熟度结论，而采用“按语义、数据模型、测试、发布、并发和故障能力逐项比较”的方法。对当前项目最稳妥的判断是：TigerFS 是 Agent workspace 语义的参考实现；通用 PGFS 仍需要独立的数据平面和服务化边界。

### 3.2 实际文件模型不是通用 inode/dentry

File-first 工作区的生成 SQL 使用合成表承载文件：

```text
id UUID PRIMARY KEY
parent_id UUID
filename TEXT
filetype TEXT  -- file / directory
body TEXT
encoding TEXT  -- utf8 / base64
created_at / modified_at
```

Markdown 工作区还会增加 `title`、`author` 和 `headers JSONB`。同一表通过 `parent_id` 和 `filename` 表示目录关系，并通过唯一约束保证同一父目录下的名字不冲突。

这套设计已经比简单的 `path + BYTEA` 更强：它支持稳定的行 ID、目录层级、目录重命名和合成格式。但它仍然是“每个工作区一个关系表”的模型，不是建议的通用 PGFS 模型中的独立：

- `inode`：对象身份、类型、模式、大小、链接数和当前内容引用；
- `dentry`：父目录与名字到 inode 的映射；
- `file_version`：文件版本到内容对象的引用；
- `object/object_chunk`：可寻址的内容和分块。

证据：`internal/tigerfs/fs/synth/build.go:34-88`、`docs/adr/017-relational-directory-structure.md`。

**判断：** TigerFS 的 `id + parent_id + filename` 是当前合成工作区的合理模型，不应误读成已经完成了一个可承载任意 Agent 文件系统的 inode/dentry 存储层。新 PGFS 若要支持 open-unlink、hard link、独立版本、快照和跨目录 O(1) rename，应把路径项和对象身份拆开。

### 3.3 目录层级和 rename 是当前实现的强项

当前仓库已经把目录关系从“字符串路径技巧”推进到了关系化模型：

- 写入文件时可自动创建父目录；
- 目录是显式记录，而不是仅由字符串前缀推导；
- 目录重命名可通过父关系和前缀更新完成，而不必把每个后代文件复制到新路径；
- FUSE 与 NFS 共享 `fs.Operations`，目录逻辑不再分散在适配器中。

这说明 TigerFS 已经验证了一个对新 PGFS 很重要的方向：**目录语义应落在数据库事务和约束中，而不是由客户端同步器拼出来。**

证据：`docs/file-first.md:115-142`、`docs/adr/009-shared-core-library.md:43-90`、`docs/adr/017-relational-directory-structure.md`。

### 3.4 History / savepoint / undo 是真实能力，但有明确边界

当前 history 的实现不是抽象设计，而是实实在在的数据库触发器和辅助表：

- 生成 `<app>_history` 历史表；
- 历史表使用 `tsdb.hypertable`，按 `version_id` 分区，并以 `file_id` 分段、按 `version_id DESC` 排序；
- BEFORE INSERT/UPDATE/DELETE trigger 保存行状态；
- `<app>_log` 记录操作；
- `<app>_savepoint` 保存命名检查点；
- 元数据表记录迁移边界，防止不安全地跨版本 undo。

源码中的 history 表明确包含 `file_id`、`parent_id`、`filename`、`body`、`version_id` 和 `operation`，并直接使用 TimescaleDB 的 `tsdb.hypertable`、`tsdb.segmentby` 和 `tsdb.orderby` 表参数；操作日志表采用同一套 hypertable + columnstore DDL。启用 `history` 的 `.build/` 操作也会先检查 `timescaledb` 扩展。

证据：`internal/tigerfs/fs/synth/build.go:288-352`、`internal/tigerfs/fs/build.go:79-91`、`docs/history.md:251-271`、`docs/adr/019-undo-boundary-via-metadata-table.md`。

这带来两个结论：

1. 分享对话中“当前 TigerFS history 依赖 TimescaleDB”的判断得到代码核实；这不是推测。更精确地说，当前实现还把 Community/TSL 的压缩路径作为设计假设，不能只看 `tsdb.hypertable` 就宣称 Apache-only。
2. “TimescaleDB 不是 PGFS 的理论必需依赖”仍然成立，但那意味着要重新设计 vanilla PostgreSQL 的版本、压缩、分区、归档和 GC 方案，而不是只替换一个扩展名。若只保留 Apache hypertable，也要证明关闭 columnstore 后的空间、写入和历史查询指标仍满足目标。

当前 history 还有已文档化的限制：data-first 直接写入 backing table 不进入同一 history/undo 体系；交错编辑下的 per-user undo 可能回退其他用户的交错修改；v0.6 到 v0.7 的迁移存在不可 undo 的边界。

证据：`docs/history.md:251-257`。

### 3.5 平台支持：macOS 可运行，但挂载后端不同

当前仓库通过 Go build tags 选择平台实现：`darwin` 选择 NFS，`linux` 选择 FUSE。macOS 路径会启动进程内 `go-nfs` 服务，然后调用 `/sbin/mount_nfs` 将本机的 NFS v3 服务挂载到目标目录；Linux 路径则直接进入 FUSE 适配器。

```text
macOS:
  macOS NFS client (`mount_nfs`)
          │ NFS v3
          ▼
  TigerFS `go-nfs` server
          │
          ▼
  shared `fs.Operations` → PostgreSQL

Linux:
  Linux kernel FUSE
          │
          ▼
  TigerFS FUSE adapter
          │
          ▼
  shared `fs.Operations` → PostgreSQL
```

因此，**TigerFS 的核心逻辑不依赖 Linux 文件系统抽象**。Linux 特有的部分主要是 `go-fuse` 和 FUSE mount；macOS 不需要 MacFUSE，也不需要加载 Linux 内核模块。仓库的共享核心负责路径解析、数据库查询、目录操作、读写、历史和控制目录，平台适配器负责把 FUSE/NFS 请求翻译为这些核心调用。

```text
Linux:  syscall → FUSE adapter → fs.Operations → PostgreSQL
macOS:  NFS RPC  → NFS adapter  → fs.Operations → PostgreSQL
```

证据：`internal/tigerfs/cmd/mount_linux.go:21-34`、`internal/tigerfs/cmd/mount_darwin.go:21-27`、`internal/tigerfs/nfs/mount_darwin.go:29-104`、`docs/adr/009-shared-core-library.md:47-90`。

仓库文档也提供了 macOS 原生运行路径：要求 macOS、Go 和可访问的 PostgreSQL；官方 Demo 使用 Docker Desktop 只运行 PostgreSQL，TigerFS 二进制本身仍在 macOS 主机上运行。Docker 不是 macOS 挂载后端的必要条件，只是 Demo 的数据库承载方式。

证据：`docs/quickstart.md:63-104`。

这对本地开发和单机 Agent 很合适，但离 SaaS 还缺少一层 Storage Service：当前 mount 进程本身就是数据库客户端、缓存持有者和文件系统协议端点。若每个短生命周期 Agent 容器都直接建立自己的连接池，连接数、鉴权、缓存失效和审计会很快成为控制面问题。

### 3.6 macOS NFS 与 Linux FUSE 的语义差异

macOS 支持并不意味着两个平台的行为完全相同。当前 macOS 后端显式使用 NFS v3，并设置 `locallocks`、`nolocks`、`rsize/wsize=128KB` 和 `noac`。这些设置反映了几个实际限制：

- `go-nfs` 不提供完整的 NFS lock manager，因此锁语义与 Linux FUSE 不同；
- NFS v3 的 RPC 是无状态的，go-nfs 需要合成 open/write/close 周期，写入可能按 RPC 分块提交；
- macOS 客户端默认的属性缓存可能看到过期的 mtime、size 或目录内容，所以当前实现关闭 NFS attribute cache，并依靠 TigerFS 自身的短 TTL stat cache；
- `wsize/rsize` 会影响大文件写入的 RPC 数和数据库提交放大，macOS 路径需要单独做性能验证。

这些差异属于**平台适配和 POSIX 语义映射问题**，不是“TigerFS 只能依赖 Linux”的证据。对用户来说，常规的 `ls/cat/grep/mkdir/mv/rm` 工作流在 macOS 上可以使用；对需要强锁、精确 open-file 生命周期或极端写入吞吐的程序，则必须把 macOS NFS 作为独立兼容性目标测试。

证据：`internal/tigerfs/nfs/mount_darwin.go:73-104`、`docs/spec.md:3253-3263`、`docs/spec.md:1696-1698`。

### 3.7 性能工程经验已经存在，且应成为新系统的起点

当前 TigerFS 已有路径、元数据和 stat 缓存，并通过 query reduction 减少远程数据库场景下的重复查询；仓库还包含针对 rename、undo、history、NFS/FUSE 和长序列操作的单测与压力测试。

但当前 macOS NFS 写入模型对远程工作区有一个重要启示：NFS v3 的 RPC 写入并不天然等于一次完整文件写入，若每个 chunk 都独立落库，会造成提交放大。仓库中的 ADR-010 已明确把“跨 RPC 持久化 memFile、close 时一次提交”列为性能设计方向。

证据：`internal/tigerfs/fs/operations.go:110-127`、`docs/adr/010-persistent-memfile.md:9-18`、`docs/adr/014-query-reduction-caching.md`、`test/stress/README.md`。

## 4. 可行性判断：PostgreSQL 适合什么，不适合什么

### 4.1 PostgreSQL 的适配点

Agent 工作区的瓶颈通常不是顺序读取几百 MB，而是大量小文件、元数据和短事务：`find`、`git status`、编辑器临时文件、lock 文件、rename-over-existing、配置文件更新等。这类 workload 需要：

- 事务和 MVCC；
- 唯一约束与外键；
- 可查询的元数据；
- 行级锁和 advisory lock；
- JSONB / BYTEA / TOAST；
- RLS、角色和审计；
- 与业务数据处于同一一致性边界。

因此 PostgreSQL 比“应用层把文件 blob 放进对象存储，再额外维护一个同步器”更适合承载**强元数据、强一致、小文件密集型工作区**。

### 4.2 PostgreSQL 不会自动提供的能力

数据库并不会因为被挂载成目录，就自动获得以下语义：

- 完整的 open file descriptor 生命周期；
- open 后 unlink 仍可读写；
- hard link、symlink、权限、xattr、ACL、mmap、sparse file；
- inotify / fswatch 等内核事件；
- 网络分区下的客户端重试和幂等；
- 大对象的经济存储和跨节点缓存；
- 语义级代码合并。

这些需要在文件系统核心、协议适配器、缓存和存储后端中明确实现或明确声明不支持。

**重要边界：** FUSE 只提供内核接口适配，不提供完整 POSIX 合规性证明；“能 mount”不等于“能跑所有编译器、包管理器和开发工具”。

## 5. 目标架构：Remote Durable Workspace

### 5.1 工作区边界

第一版只把 Agent workspace 虚拟化，不把整个容器文件系统塞进 PostgreSQL：

```text
Agent container
├── image /usr /bin /etc       → 镜像只读层
├── /tmp /cache                → 本地 ephemeral disk
├── /dev /proc /sys /socket    → 容器和内核
├── secrets                    → Secret Manager
├── /workspace                 → Remote Durable Workspace
└── 巨型 artifact / dataset    → 后续 Object Storage
```

这样可以避免把设备文件、socket、系统目录和编译临时文件都纳入分布式文件系统，显著缩小兼容性和安全范围。

### 5.2 建议的生产拓扑

```text
             Control Plane
       identity / workspace lease
                    │
                    ▼
Agent container ── local client / mount
       /workspace       │
                         │ FUSE, NFS, CSI or native client
                         ▼
                 PGFS Gateway / Storage Service
                 ├── AuthZ and tenant context
                 ├── path and metadata cache
                 ├── request idempotency
                 ├── transactions and locks
                 ├── version / snapshot / undo
                 ├── quota and audit
                 └── pooled PostgreSQL connections
                         │
                         ▼
                PostgreSQL core + optional object store
```

开发 PoC 可以先做 `FUSE → PostgreSQL`，但 SaaS 生产形态应收敛为：

```text
Agent → thin client / node adapter → Storage Service → PostgreSQL
```

这样 5,000 个 Agent 不会自然演化成 5,000 个数据库连接池，也能把鉴权、租户隔离、限流和审计集中在一个受控服务中。

### 5.3 FUSE、NFS、CSI 的角色

- **FUSE**：Linux 本地兼容层，适合 PoC、开发机和节点级挂载。
- **NFS**：macOS 或某些受限客户端的适配层；当前 TigerFS 在 macOS 上使用系统 `mount_nfs`，不依赖 MacFUSE；必须特别关注 NFS cache、写入分块和 close/commit 语义。
- **CSI**：Kubernetes 生产部署的自然接口，但 CSI 只解决卷的生命周期和挂载，不替代 PGFS 的租户、数据模型和一致性层。
- **HTTP/gRPC/native API**：浏览器、Web IDE、CI、远程 Agent 等非 POSIX 客户端的直接接口，应与文件系统适配器共享同一个服务核心。

FUSE/CSI/NFS 都不应成为租户安全边界。尤其不要默认给不可信 Agent 容器 `SYS_ADMIN`、`/dev/fuse` 或 mount namespace 控制权；更安全的方向是节点侧完成挂载，再将已授权的工作区绑定进容器。

对于当前仓库的 macOS 用户，最直接的验证命令是：

```bash
# 使用 Demo：PostgreSQL 在 Docker，TigerFS 原生运行在 macOS
./scripts/demo/demo.sh start --mac

# 或连接已有 PostgreSQL
./bin/tigerfs mount postgres://user:password@host:5432/db /tmp/tigerfs-demo
```

这条路径要求主机具备 macOS 的 `/sbin/mount_nfs` 和可访问的 PostgreSQL；它不要求 Linux FUSE。当前仓库的 `go.mod` 要求 Go `1.25.1`，编译前应使用满足该版本要求的 Go 工具链。

## 6. 建议的数据模型

### 6.1 为什么不直接使用 `files(path, data BYTEA)`

单表路径模型很快会在以下操作上失去清晰语义：

- rename 不应复制整棵目录树；
- hard link 不应复制文件内容；
- version 不应复制完整 blob；
- undo 不应把其他 Agent 的修改静默覆盖；
- snapshot/fork 不应复制整个 workspace；
- open 后 unlink 需要让 inode 在无路径时继续存活。

因此建议把“名字”“对象身份”“内容版本”和“操作历史”分离。

### 6.2 最小逻辑模型

```text
tenant
  └── workspace
        └── root inode
              ├── dentry(name → child inode)
              ├── file_version
              └── content object / chunks

workspace transaction
  ├── operation log
  ├── savepoint
  └── snapshot / GC metadata
```

建议的核心实体：

- `tenant`：计费、安全和生命周期的最小边界；
- `workspace`：Agent 看到的持久化工作区，是控制面分配的第一等对象；
- `inode`：稳定对象身份、类型、mode、size、nlink、mtime、当前版本引用；
- `dentry`：`(parent_inode_id, name) → child_inode_id`，并以唯一键防止同目录冲突；
- `file_version`：inode 的逻辑版本和内容对象引用；
- `object`：内容 hash、大小、inline data 或 chunk manifest；
- `object_chunk`：大文件的分块内容；
- `operation`：只记录元数据变更和对象引用，不重复保存大文件内容；
- `workspace_txn`：将多次文件操作归入一个可审计事务组；
- `savepoint`：将工作区 generation / operation sequence 固化为回滚锚点；
- `quota/gc_queue`：分别管理逻辑配额和不可达内容回收。

### 6.3 内容存储策略

建议先采用混合策略，不把所有内容都立即引入对象存储：

- 小文件内联在 `object` 的 `bytea` 字段中，减少 round trip；
- 中型文件使用固定大小 chunk，先从 4 MiB 左右开始基准测试；
- 超大 artifact 交给对象存储，数据库只保存 manifest、hash 和租户归属；
- 内容寻址可以消除同一租户内重复版本的内容复制，但默认不要跨租户去重。

“同一内容只存一次”不是免费收益：hash 计算、GC、租户隐私、加密密钥和引用计数都会随之进入系统。V1 先做租户内 dedup，跨租户 dedup 作为明确的后续决策。

这也是 PostgreSQL 与对象存储之间应采用混合而不是二选一的原因：对象存储擅长海量大对象和低成本 artifact，但单独使用它无法自然提供目录唯一性、原子 rename、细粒度 metadata、事务和可查询审计；PostgreSQL 适合工作区的元数据与小文件热路径，却不应被强行用作所有大型 dataset 的最终对象仓库。

内容后端的选择应按 workload 分层：

- 小文件和频繁修改文件优先走 PostgreSQL inline/TOAST；
- 需要局部读写但不适合单列存储的文件走 chunk table；
- 大型构建产物、模型文件和数据集走对象存储，PGFS 只保存 manifest、hash、版本和权限元数据。

### 6.4 TigerFS 与新模型的关系

TigerFS 当前的合成表可视为“workspace adapter / compatibility backend”，而不是新 PGFS 的最终物理模型：

```text
WorkspaceFS semantics
        │
        ├── PGFS inode/dentry backend   ← 新的通用工作区
        ├── TigerFS synth-table backend ← 兼容现有 file-first
        └── PostgreSQL data-first view  ← 数据库映射能力
```

这样既能复用 TigerFS 的 Agent-friendly 文件接口，又不会强迫 data-first 的关系表映射承担任意二进制文件系统的全部语义。

## 7. 一致性、事务和恢复

### 7.1 事务边界

建议区分三种边界：

1. **单 syscall 事务**：`mkdir`、`rename`、`unlink` 等每个 mutation 至少是一个数据库事务。
2. **文件句柄事务**：连续 `write()` 先进入客户端或 gateway buffer，在 `fsync/close` 时形成一次或少数几次提交。
3. **工作区事务**：为高级 Agent API 提供多文件原子提交，例如一次修改源码、测试和配置，然后统一 commit。

不要把“每个网络包”当成事务边界，也不要承诺普通 FUSE 的多个 syscall 自动构成跨文件原子事务；需要显式 API 和生命周期才能做到这一点。

### 7.2 Rename 和目录环检测

rename 应锁住 source inode、source parent 和 target parent，按稳定顺序加锁以减少死锁；用 `UNIQUE(parent_inode_id, name)` 作为最终竞争保证，而不是先 `SELECT` 再 `INSERT`。

目录 rename 不应递归更新所有后代；只需改变目标目录的 dentry。目标 parent 不能属于 source subtree，V1 可用 recursive CTE 做环检测，后续再评估 `ltree`、closure table 或 materialized path。

### 7.3 Undo 必须是条件恢复

当前 TigerFS 文档已经承认交错编辑时 per-user undo 的限制：恢复某个用户的变化可能同时回退其他用户在中间插入的编辑。新 PGFS 不应简单复制“读取 before_state，然后覆盖回去”的策略。

建议每个可恢复状态携带版本号，并采用 compare-and-swap：

```text
Agent A: version 10 → 11
Agent B: version 11 → 12
Agent A: undo version 11
         WHERE current_version = 11
         → 0 rows, report conflict
```

undo 的正确结果应是“冲突可见、数据不被静默覆盖”，而不是假装回到了历史状态。

### 7.4 请求幂等与故障语义

所有 mutation RPC 都应带 `request_id`。典型故障是：数据库 commit 成功，网络响应在返回前丢失，客户端随后重试。如果没有幂等键，可能出现重复创建、重复 rename 或重复写入。

Storage Service 应保留最近的请求结果或使用唯一约束，使以下重试安全：

```text
same request_id + same workspace + same actor
        → return original result
```

还需要明确：

- `write()` 成功是“已被服务接受”还是“已持久化”；
- `fsync()` 是否等待数据库 commit；
- close 失败时文件句柄和 dirty buffer 如何处理；
- network timeout 后客户端如何查询请求状态；
- PostgreSQL 重启、Gateway 重启、Pod 迁移后是否能恢复未完成句柄。

建议默认语义为：`write-visible, fsync-durable`，并将更强的确认模式作为显式选项。

### 7.5 Snapshot、rollback 和 GC

V1 可以用 `file_version + operation + savepoint` 完成小规模回滚；当操作日志达到百万级别后，逐条反向 replay 会不可接受。下一阶段应增加 materialized workspace snapshot：保存 inode/dentry 当前状态和对象引用，回滚时从最近快照开始重放少量操作。

若内容对象采用 hash，snapshot/fork 可以主要表现为根指针或版本引用，而不是复制文件。但对象 GC 必须区分：当前引用、历史保留、快照引用、正在写入和待确认的幂等请求。

### 7.6 配额、热点工作区与读副本

配额至少要区分三种量：

- **logical bytes**：用户看到的文件总大小；
- **physical bytes**：去重、压缩和 chunk 后实际占用；
- **file/operation counts**：文件数、目录数、历史操作数和并发句柄数。

配额检查必须与 mutation 放在同一事务中，不能先在缓存中估算再异步修正，否则并发 Agent 可以同时越过配额。多租户规模扩大后，也不应默认“一租户一个 PostgreSQL database”；应按 workspace 热度、数据大小、隔离要求和运维成本选择 schema、分片或数据库级隔离。

读副本不能直接承接普通 filesystem read。写入成功后，客户端可能立刻读取刚写入的文件；若后续使用副本，应记录 commit LSN，并允许读取端等待副本 replay 到对应 LSN，或者在一致性窗口内强制走主库。

## 8. 多租户与安全模型

### 8.1 租户身份不能由路径决定

不应把 URL 中的 `tenant_id` 直接当作数据库安全上下文。正确流程是：

```text
JWT / session / workload identity
        ↓
trusted authorization context
        ↓
tenant_id + workspace_id + actor_id + permissions
        ↓
Storage Service
```

路径中的 workspace 名称只用于查找，不用于决定调用者是谁。这能避免 confused deputy：调用者不能通过修改路径参数切换到另一个租户。

### 8.2 PostgreSQL RLS 的使用边界

推荐让所有租户数据都通过 `workspace → tenant` 关联，并在关键表上显式启用 RLS。每个请求在事务内使用 `SET LOCAL app.tenant_id = ...`，不要把 session 级状态留在连接池里。

生产部署还应注意：表 owner 通常可绕过 RLS，因此 Storage Service 应使用独立运行角色，并按需要 `FORCE ROW LEVEL SECURITY`。RLS 是数据库层的纵深防御，不替代上层授权、workspace lease 和审计。

### 8.3 FUSE 不应成为安全边界

把 `/dev/fuse` 和 mount 能力交给不可信代码，会扩大容器突破和跨挂载访问的风险面。建议：

- 开发机 PoC 可以直接本地 FUSE；
- SaaS 中由节点侧或受控 sidecar 挂载；
- Agent 容器只获得绑定后的 `/workspace`；
- Storage Service 对每个 RPC 校验 tenant/workspace/actor；
- 数据库再用 RLS 和最小权限角色兜底。

## 9. 缓存与性能模型

### 9.1 性能关键不是大文件吞吐，而是 metadata amplification

最危险的路径不是“读 10 MB”，而是：

```text
find project
  → 数万次 lookup/stat
  → 数万次 RPC
  → 数万次 SQL
```

因此第一天就应把“一个文件系统 workload 触发多少 RPC、SQL 和 commit”设为核心指标，而不是只测顺序读吞吐。

### 9.2 三级缓存

- **L1 path/inode cache**：`path → inode_id`，短 TTL，降低重复 lookup。
- **L2 metadata/directory cache**：inode stat 和整目录 child 列表，一次 SQL 服务多个 `readdir/lookup/stat`。
- **L3 content/chunk cache**：按 `inode + version` 缓存热点小文件，按 chunk 缓存大文件。

TigerFS 当前已经有有限 TTL 的 stat/metadata 缓存和查询削减经验，可作为基线，但新 PGFS 的多节点缓存必须加入 generation 和失效机制。

### 9.3 失效不能只靠 NOTIFY

写事务可增加 workspace generation，并发送 PostgreSQL `NOTIFY` 作为失效提示：

```text
transaction commit
  ├── workspace.generation++
  └── NOTIFY workspace channel
```

`NOTIFY` 不是可靠消息总线。Gateway 断线后可能漏通知，因此恢复时必须重新读取 generation 并丢弃不确定缓存。V1 不必引入 Kafka/NATS；可靠事实来源仍是 PostgreSQL，通知只负责降低过期窗口。

### 9.4 NFS / FUSE 写缓冲

写入应以文件句柄或连续写入窗口为单位缓冲，避免每个 32 KiB NFS chunk 都触发完整 SQL commit。缓存必须有：

- 最大文件和总内存上限；
- close/fsync/超时/重命名/删除时的 flush 规则；
- 崩溃后 dirty data 的明确语义；
- open-unlink 句柄的引用计数；
- 大文件流式写入，避免单个 RPC 携带巨型 byte array。

## 10. Agent 文件系统兼容性目标

第一版应定义一个明确的 `Agent Filesystem Compatibility Set`，而不是笼统宣称“POSIX-compatible”。

### L0：必须支持

- `open/read/pread/write/pwrite/close`
- `create/truncate/mkdir/rmdir/readdir`
- `stat/lstat`
- `rename/unlink`
- `fsync`

### L1：应尽快支持

- rename-over-existing 和 atomic replace；
- `chmod`、mtime、临时文件；
- symlink / readlink；
- `flock` 或至少可诊断的 lock 语义；
- hard link；
- open 后 unlink；
- 足够稳定的目录 mtime 和文件大小。

### L2：以后再做或明确不支持

- `mmap`、xattr、ACL；
- device node、socket、FIFO、ioctl；
- sparse file；
- inotify 的完整行为；
- 内核级特殊文件语义。

### 兼容性矩阵

至少应测试：

- 基础工具：`touch/mkdir/rmdir/cp/mv/rm/cat/dd/truncate/chmod/ln`；
- 搜索工具：`find/grep/rg/git grep`；
- Git：`init/add/commit/status/diff/checkout/switch/merge/reset/stash`；
- 包管理器：`npm/pnpm/yarn/pip/uv/poetry/cargo/go/maven/gradle`；
- Agent：Codex、Claude Code、Aider、OpenHands 和自研 Agent；
- 编辑器模式：临时文件写入、fsync、rename 替换、并发保存；
- 故障：kill Agent、kill client、kill Gateway、数据库断线、Pod 重启、网络分区、重复 RPC。

每个 workload 不只记录成功/失败，还应记录：

- filesystem syscall 数；
- RPC 数与重试数；
- SQL 数、SQL 延迟、连接池等待；
- cache hit/miss；
- fsync 延迟；
- CAS/undo 冲突；
- 数据损坏、幽灵文件、租户越界等正确性结果。

第一版性能目标应作为工程目标而不是既定事实，并且优先关注 metadata amplification：例如 warm metadata lookup、warm small-file read 和 cold metadata operation 的 p95 延迟，以及 `readdir(10k files)` 是否保持 O(1) 到 O(batch) 的数据库往返，而不是产生 10,000 次 SQL。具体阈值应在真实 Agent workload 和部署拓扑确定后再冻结。

## 10.1 共享工作区、协作与 Git 的边界

同一个 workspace 可以同时服务人类、多个 Agent、CI 和 Web IDE；这能消除“每个参与者一份本地 clone，再靠同步器合并”的一部分复杂度。但共享不等于自动语义合并：

- PGFS 负责 atomic operation、版本、冲突检测、不可损坏和可回滚；
- PGFS 不负责理解 Python/Go/JavaScript 的语义并自动合并代码；
- Git 仍然负责 branch、commit、merge、review 等开发者源代码管理能力。

更合理的关系是：PGFS 作为运行时持久工作区，工作区内可以包含 `.git/`；Git commit 可以成为业务层 checkpoint，而不是被 PGFS 取代。

## 11. TigerFS 之外需要补齐的产品能力

当前 TigerFS 的强项集中在“数据库和合成工作区的文件接口”。要成为 SaaS 级 Remote Durable Workspace，还需要新的产品层：

### 11.1 Control Plane

- workspace 创建、销毁、fork、lease 和生命周期；
- 将 user/session/workload identity 绑定到 tenant/workspace；
- 配额、计费、区域、保留策略；
- 将短生命周期 Agent 重新连接到同一 workspace；
- 管理 workspace snapshot、备份和恢复。

### 11.2 Storage Service

- 薄协议适配器背后的统一 filesystem semantics；
- 连接池复用和数据库限流；
- request idempotency、超时和重试；
- RLS context、审计和 quota enforcement；
- metadata/content cache；
- 版本、回滚、GC 和后台 compaction。

### 11.3 Data Plane

- vanilla PostgreSQL 核心 schema；
- 可选 TimescaleDB history/analytics；
- 可选对象存储 backend；
- 可选本地 backend，方便测试和离线开发；
- 未来的 read replica、sharding 和多区域复制。

把这三层混在一个 FUSE 进程中，会使开发环境的简单性变成生产环境的耦合。TigerFS 的共享核心是好的方向，但 PGFS 还需要把“挂载”“存储服务”“控制面”分成不同的可演进边界。

## 12. 关于 TimescaleDB：技术依赖与商业风险分离处理

### 12.1 官方 edition matrix 的可核查边界

截至本报告日期，Timescale 官方 edition matrix 给出的边界可以概括为：

- **Apache 2 Edition**：`CREATE TABLE` hypertable 语法、`create_hypertable`、`show_chunks`、`drop_chunks`、维度管理、tablespace 管理和基础 hypertable 信息视图等基础能力可用；
- **Community Edition（含 Timescale License/TSL 部分）**：Hypercore/columnstore、continuous aggregates、data-retention policy、jobs/automation、SkipScan，以及 `split_chunk`、`reorder_chunk`、`move_chunk` 等高级能力可用；
- 因而“hypertable 在 Apache、所有压缩/历史能力都在 TSL”是过度简化。应按具体 API 和版本逐项核对；当前官方矩阵明确把 Hypercore/columnstore 与上述高级能力标为 Community，而不是把所有涉及压缩的旧 API 统称为同一类。

官方资料：

- [Compare TimescaleDB editions](https://www.tigerdata.com/docs/get-started/choose-your-path/timescaledb-editions)
- [TimescaleDB Apache license](https://github.com/timescale/timescaledb/blob/main/LICENSE-APACHE)
- [Timescale License agreement](https://github.com/timescale/timescaledb/blob/main/tsl/LICENSE-TIMESCALE)

### 12.2 当前 TigerFS 代码事实

当前 TigerFS history 的创建流程会检查 TimescaleDB，并生成使用 `tsdb.hypertable`、`tsdb.segmentby`、`tsdb.orderby` 的历史表和操作日志表。源码注释把这套 DDL 明确描述为“hypertable + columnstore”，并依赖自动 columnstore/compression policy。CI 也使用 TimescaleDB PostgreSQL 镜像执行测试。需要注意，当前前置检查只判断 `timescaledb` 扩展是否存在，并不检查 `timescaledb.license`；因此 Apache 模式可能通过扩展检查，却在后续以关闭压缩的状态继续建表。

证据：`internal/tigerfs/fs/build.go:79-91`、`internal/tigerfs/fs/synth/build.go:335-352`、`internal/tigerfs/fs/synth/build.go:439-456`、`.github/workflows/test.yml:21-28`。

因此，“现有 history 可以直接在 vanilla PostgreSQL 上运行”不是当前仓库事实，不能在产品设计里默认成立。

### 12.3 对 edition 行为的本机探针

使用仓库测试所需的 `timescale/timescaledb-ha:pg18`（TimescaleDB `2.30.2`、PostgreSQL `18.6`）做了最小 SQL 探针：同一张表使用 TigerFS 当前的 `CREATE TABLE ... WITH (tsdb.hypertable, tsdb.segmentby, tsdb.orderby)` DDL。

结果：

| 模式 | hypertable DDL | `compression_enabled` | 解释 |
| --- | --- | ---: | --- |
| Community/默认 `timescale` | 成功 | `true` | 得到当前 TigerFS 所期待的压缩/columnstore 状态 |
| Apache `apache` | 成功 | `false` | 基础 hypertable 仍可用，但不能把 DDL 能执行误认为已获得同等 columnstore 能力 |

此外，Apache 模式仍可创建不带 columnstore 依赖的基础 hypertable。这正好验证了官方 matrix 的分层：**Apache hypertable 能力保留，Community columnstore 能力不等价保留。** 完整的 TigerFS history 集成测试使用的是默认 Community 模式；此前 vanilla PostgreSQL 16 的测试则在更早的 extension 检查阶段失败。

探针过程与完整构建/测试记录见 `docs/drafts/tigerfs-build-test-log-2026-10-01.md` 的 3.8 节。

### 12.4 对分享回复的逐项判断

分享回复的核心方向是对的，但需要作以下校正：

1. **“新 PGFS 不必依赖 TSL”——作为目标架构，正确。** 普通表、事务、MVCC、`bytea`/TOAST、索引、RLS、advisory lock，以及 Apache hypertable（若确实需要）都可以构成不依赖 Hypercore 的底座。
2. **“当前 TigerFS 不需要 TSL”——按当前代码，不能成立。** 当前 history/log DDL 明确追求 columnstore/compression；Apache 模式虽然接受相同建表语法，但本机探针显示压缩状态为关闭，所以这不是功能等价的 Apache 运行路径。
3. **“continuous aggregates、retention policy、chunk move/split 等不是 TigerFS 核心必需”——基本正确。** 当前仓库的 history 依赖点是 hypertable 与 columnstore/compression DDL，并没有把 continuous aggregates、retention policy 或高级 chunk 管理 API 写入 TigerFS 核心路径。它们仍可作为未来 telemetry/analytics 优化，但不应被误写成现有实现已经依赖的组件。
4. **“TSL 一定不能用于 SaaS”——方向正确但表述过于绝对。** 当前 TSL 文本禁止直接以 TSL 软件提供 time-sharing、database-as-a-service 或向第三方提供数据库操作的 SaaS，但对作为 Value Added Product/Service 的组合方式设有条件性许可和客户通知/结构限制。TigerFS 若要做多租户 SaaS，必须按拟采用的 Timescale 版本、是否暴露数据库接口、是否允许客户定义 schema，以及是否分发二进制逐项做法律审查，不能只引用一句“不能卖服务”。

### 12.5 对新 PGFS 的建议

新 PGFS 应把这两个问题分开：

- **技术层**：原生 PostgreSQL 的 MVCC、事务、分区、JSONB、BYTEA/TOAST、索引、RLS、advisory lock 是否足以承载 V1 history？答案是可以作为设计目标，但必须重新实现和基准测试。
- **许可层**：若只使用 Apache 2 Edition 的 hypertable 基础能力，许可证边界相对清晰；若使用 Community/TSL 的 columnstore、continuous aggregates、retention/jobs 或其他受限模块，部署和提供 SaaS 时是否触发 TSL 条件，必须按具体版本和分发方式做法律审查。

最稳妥的架构是：

```text
PGFS core
  └── vanilla PostgreSQL
       ├── filesystem semantics
       ├── tenant isolation
       ├── version / snapshot
       ├── quota / audit
       └── rollback

Optional accelerators
  └── TimescaleDB
       ├── Apache hypertable partitioning
       ├── optional Community/TSL columnstore compression
       ├── analytics
       └── telemetry / retention jobs
```

如果继续沿用 TigerFS 的 history 实现，应把“TimescaleDB Community-enabled deployment”及对应 TSL 条款明确写进部署前置条件和商业许可清单；如果要做更干净的 SaaS 核心，则应先实现并基准测试 vanilla PostgreSQL 或 Apache-only history backend，再把 Community/TSL columnstore 作为显式可选 profile，而不是在运行时静默依赖。

## 13. 分阶段实施路线

### Phase 0：定义产品边界

交付物：

- `Agent Filesystem Compatibility Set`；
- workspace、tenant、actor、session 的身份模型；
- write-visible / fsync-durable 语义；
- 对 binary、symlink、hardlink、mmap、watch 的支持矩阵；
- TigerFS synth backend 与新 PGFS backend 的关系。

通过标准：所有“不支持”的语义都有可观察错误，而不是静默近似。

### Phase 1：最小 WorkspaceFS PoC

只实现：

```text
lookup, readdir, open, read, write,
mkdir, rename, unlink, stat, truncate, fsync
```

建议：

- Linux FUSE 适配；
- PostgreSQL 单库、单 workspace；
- `inode + dentry + object` 最小 schema；
- 无多节点、无 CSI、无 TimescaleDB；
- 允许先用单 chunk 或小文件 inline；
- 运行真实 Codex、Claude Code、Git、Node/Python 项目。

平台策略：先在 Linux FUSE 和 macOS NFS 上各跑一套最小兼容性矩阵。Linux 结果不能替代 macOS 验证，macOS 结果也不能替代 Linux FUSE 验证；尤其要单独覆盖 NFS 的分块写、属性缓存、锁和 close/flush 语义。

这一步只回答三个高价值问题：

1. Agent 能否不改代码运行？
2. metadata amplification 是否能通过批量查询和缓存控制？
3. 哪些 L1 语义实际上是第一天必需？

### Phase 2：Storage Service 化

将本地 FUSE 的数据库访问抽到 gRPC/HTTP Storage Service：

- request idempotency；
- shared connection pool；
- tenant/workspace/actor authorization；
- `SET LOCAL` + RLS；
- batch lookup/readdir；
- metadata generation 和失效；
- 基础审计、限流和指标。

这一阶段完成后，才有资格声称“计算与工作区解耦”。

### Phase 3：版本、快照和恢复

- file version 和 content object；
- operation log 和 savepoint；
- CAS undo；
- materialized workspace snapshot；
- crash recovery、GC 和保留策略；
- 多 Agent 共享工作区的冲突检测。

### Phase 4：生产化部署

- 节点级 mount 或 CSI；
- Pod 重启和 workspace reattach；
- quota、计费和租户级指标；
- 对象存储混合后端；
- read replica 的 LSN-aware 读取策略；
- 分片、热点 workspace 和多区域复制。

CSI 不应提前于 Phase 2/3。否则只是把不明确的文件系统语义包装成 Kubernetes volume，并没有解决核心一致性和安全问题。

## 14. Go / No-Go 判定标准

在进入大规模分布式实现前，建议以以下标准作为闸门：

### Go 条件

- Codex、Claude Code 和普通 Git 项目可在不改 Agent 代码的情况下完成创建、读取、编辑、测试和提交；
- 编辑器的 temp-write → fsync → atomic rename 模式不会损坏或丢失文件；
- `find`、`git status` 和 package install 的 SQL/RPC 数量在缓存命中时可控；
- 数据库重启、Gateway 重启和重复 RPC 不造成重复或幽灵状态；
- 两个 Agent 的冲突能被检测、记录和恢复，而不是静默覆盖；
- 租户边界有自动化负向测试，无法通过路径或缓存访问其他租户；
- workspace 删除、重新连接和 snapshot restore 的生命周期闭环成立。

### No-Go 条件

- 仍需改写每个 Agent 的文件 API；
- 主要 benchmark 只测顺序大文件读取；
- 依赖给不可信容器开放 mount 特权；
- 没有 request idempotency 和超时后的状态查询；
- undo 仍可能静默抹掉其他 Agent 的修改；
- TimescaleDB、Object Storage、PostgreSQL 的许可/部署边界没有书面确认；
- 只证明单机可挂载，却没有跨进程、跨节点和故障恢复测试。

## 15. 主要风险与缓解措施

### 风险一：把“FUSE 可挂载”误认为 POSIX 兼容

缓解：发布兼容性矩阵和负向测试；把 unsupported 语义显式化。

### 风险二：元数据放大导致系统不可用

缓解：目录批量读取、stat 聚合、三级缓存、SQL/RPC/commit 计数，以及真实 Agent workload 基准。

### 风险三：缓存失效造成跨 Agent 读到旧数据

缓解：generation 是事实来源，NOTIFY 只是提示；断线恢复时丢弃不确定缓存；内容缓存键必须包含版本。

### 风险四：回滚覆盖并发修改

缓解：CAS undo、冲突报告、workspace transaction 和显式恢复策略。

### 风险五：租户身份泄漏到连接池或缓存键

缓解：每个事务 `SET LOCAL`，使用独立运行角色和 RLS，缓存键强制带 tenant/workspace，做跨租户负向测试。

### 风险六：历史存储成本失控

缓解：内容对象与 operation metadata 分离；按租户/工作区配置 history retention；快照减少 replay；GC 延迟回收并可审计。

### 风险七：过早引入 CSI、分片和多区域

缓解：以最小 FUSE PoC 验证 Agent 兼容性；每次只增加一个分布式复杂度来源，并保留可回放的 workload。

## 16. 结论

这次对话最值得保留的不是“PostgreSQL 可以模拟目录”这一事实，而是下面的架构判断：

```text
Agent thinks it uses a local filesystem
                    │
                    ▼
          WorkspaceFS compatibility ABI
                    │
                    ▼
           Remote Durable Workspace
                    │
                    ▼
        PostgreSQL today, hybrid later
```

TigerFS 当前版本已经提供了重要的验证基础：

- PostgreSQL 作为文件/数据库接口的产品形态；
- File-first 与 data-first 双模式；
- 共享 `fs.Operations` 核心；
- `parent_id` 目录层级和原子重命名；
- history、log、savepoint、undo；
- macOS 原生 NFS 与 Linux FUSE 两条挂载路径；
- query reduction、缓存和真实压力测试；
- Linux FUSE 与 macOS NFS 适配器分层。

但 TigerFS 当前实现也清楚地告诉我们下一阶段不能直接“把一个 mount 进程搬到云上”：

- File-first 的主载荷仍是文本/合成文件，不是任意二进制 POSIX 盘；
- history 当前实际依赖 TimescaleDB；
- 进程内 mount 不是多租户 Storage Service；
- 当前 undo 语义仍需在并发共享工作区中升级为条件恢复；
- 多节点缓存、幂等 RPC、RLS、配额、对象 GC 和 workspace 生命周期仍属于新产品层。

因此最稳妥的实施路径是：**以 TigerFS 作为语义和测试参考，以新的 `WorkspaceFS / PGFS` 核心作为 SaaS 边界；先用极小 PoC 验证真实 Agent，再逐步引入服务化、版本化、租户化和 CSI。**

## 附录 A：已核查的仓库证据

- 产品定位和双模式：`README.md:3-28`、`docs/spec.md:41-57`。
- File-first 表结构：`internal/tigerfs/fs/synth/build.go:34-88`。
- History 表和 Timescale hypertable：`internal/tigerfs/fs/synth/build.go:288-352`。
- History/log 的 columnstore 配置：`internal/tigerfs/fs/synth/build.go:335-356`、`internal/tigerfs/fs/synth/build.go:439-456`。
- History 启用前的 TimescaleDB 检查：`internal/tigerfs/fs/build.go:79-91`。
- 共享核心与 FUSE/NFS 适配边界：`docs/adr/009-shared-core-library.md:43-90`、`internal/tigerfs/cmd/mount_linux.go:21-34`、`internal/tigerfs/cmd/mount_darwin.go:21-27`。
- macOS 原生 NFS 挂载：`internal/tigerfs/nfs/mount_darwin.go:29-104`、`internal/tigerfs/nfs/mount_linux.go:12-24`。
- macOS 运行方式和 Demo：`docs/quickstart.md:63-104`、`scripts/demo/demo.sh:217`。
- History、savepoint、undo 的限制：`docs/history.md:251-271`。
- 目录关系和 rename 设计：`docs/file-first.md:115-142`、`docs/adr/017-relational-directory-structure.md`。
- 查询削减、缓存与 NFS 写入问题：`internal/tigerfs/fs/operations.go:110-127`、`docs/adr/010-persistent-memfile.md`、`docs/adr/014-query-reduction-caching.md`。
- 压力测试范围和故障诊断：`docs/adr/018-stress-test.md`、`test/stress/README.md`。
- CI 使用 TimescaleDB：`.github/workflows/test.yml:21-28`。

## 附录 B：外部资料入口

以下资料用于核对 PostgreSQL 行级安全、对象存储能力、锁和适配器边界；报告中的外部结论应以相应版本的官方文档为准：

- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [PostgreSQL Large Objects](https://www.postgresql.org/docs/current/largeobjects.html)
- [PostgreSQL TOAST](https://www.postgresql.org/docs/current/storage-toast.html)
- [PostgreSQL Explicit Locking / Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
- [libfuse](https://github.com/libfuse/libfuse)
- [Container Storage Interface specification](https://github.com/container-storage-interface/spec)
- [TimescaleDB license file](https://github.com/timescale/timescaledb/blob/main/LICENSE)
- [TimescaleDB edition comparison](https://www.tigerdata.com/docs/get-started/choose-your-path/timescaledb-editions)
- [Timescale License agreement](https://github.com/timescale/timescaledb/blob/main/tsl/LICENSE-TIMESCALE)
- [TigerFS repository](https://github.com/wubuku/tigerfs)
- [分享对话](https://chatgpt.com/share/6abe4255-5d1c-83e8-aee3-9808a60b9c0a)

## 附录 C：本机验证记录

本报告中的 macOS/NFS 可运行性判断已在本机使用 Docker PostgreSQL 进行验证：Go `1.25.14 darwin/arm64` 可以完成 `go build ./...`，内部包测试通过；先用 vanilla PostgreSQL 16 验证了 CRUD、目录、NFS 大文件和部分 File-first workspace，再通过国内镜像拉取并重标记 `timescale/timescaledb-ha:pg18`，使完整集成测试成功通过。完整过程、数据库版本、代理设置、镜像重标记命令和环境边界，见 `docs/drafts/tigerfs-build-test-log-2026-10-01.md`。
