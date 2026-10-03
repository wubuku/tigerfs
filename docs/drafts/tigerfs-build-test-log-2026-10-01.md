# TigerFS 本机 macOS 构建与测试记录

- 日期：2026-10-01
- 主机：macOS，Apple Silicon（`darwin/arm64`）
- 仓库：`/Users/yangjiefeng/Documents/wubuku/tigerfs`
- 目的：验证当前 Go 工具链、Docker PostgreSQL 和 macOS 原生 NFS 路径能否构建并运行 TigerFS。
- 记录性质：一次可复核的本机验证记录；不是生产环境兼容性承诺。

## 1. 测试环境

### 1.1 Go 工具链

```text
go version go1.25.14 darwin/arm64
/opt/homebrew/bin/go
```

仓库 `go.mod` 声明 `go 1.25.1`，当前 Go `1.25.14` 满足该要求。此前机器上存在 `go1.25.0 darwin/amd64`，本次使用 Homebrew 原生 Apple Silicon 工具链替换。

### 1.2 Docker 与 PostgreSQL

本机 Docker/OrbStack 可用：

```text
Docker Server: 29.4.0
Docker Compose: v5.1.2
```

本次使用已经运行的 Docker 容器 `postgresql`，其宿主机端口映射为 `127.0.0.1:5432`，容器镜像为 `postgres:16`。连接信息按用户提供的配置使用，但本文不重复记录密码：

```text
host: 127.0.0.1
port: 5432
database: tigerfs_test_20261001
username: postgres
```

数据库连接验证结果：

```text
PostgreSQL 16.8 (Debian 16.8-1.pgdg120+1) on aarch64-unknown-linux-gnu
current_user: postgres
current_database: tigerfs_test_20261001
```

数据库 `tigerfs_test_20261001` 是本次单独创建的隔离数据库；测试代码为每个测试创建并清理独立 schema。

### 1.3 外网代理

下载 Go 模块时使用了用户提供的本机代理：

```bash
export https_proxy=http://127.0.0.1:9981
export http_proxy=http://127.0.0.1:9981
export all_proxy=socks5://127.0.0.1:9981
export GOPROXY=https://proxy.golang.org,direct
```

代理用于外网资源下载；连接本机 PostgreSQL 时仍直接访问 `127.0.0.1:5432`。

## 2. 构建过程

### 2.1 下载依赖

执行：

```bash
go mod download
```

结果：成功。首次下载约耗时 1 分钟；此前不带代理执行时，依赖下载在 `github.com/klauspost/compress` 等模块处长期无进展。

### 2.2 构建全部 Go 包

执行：

```bash
go build ./...
```

结果：成功，退出码 `0`，耗时约 `1.4s`。

## 3. 测试过程

### 3.1 测试数据库准备

用 Docker 中的 PostgreSQL 创建测试数据库：

```sql
CREATE DATABASE tigerfs_test_20261001;
```

仓库集成测试会尝试启用 `timescaledb`。当前 `postgres:16` 容器没有该扩展，但有 `uuid-ossp`。为了隔离验证不依赖历史功能的文件系统路径，本次只在测试数据库中增加了临时兼容函数：

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE OR REPLACE FUNCTION public.uuidv7()
RETURNS uuid
LANGUAGE sql
VOLATILE
AS $$ SELECT uuid_generate_v4(); $$;
```

这个函数只是把 PostgreSQL 16 的缺失函数映射到随机 UUID，**不等价于真正的 UUIDv7，也没有修改仓库代码**。它只能帮助验证目录、NFS、读写等不依赖 UUIDv7 排序语义的路径；涉及 history/TimescaleDB 的测试仍必须使用 TimescaleDB-enabled PostgreSQL。

### 3.2 测试列表发现

执行：

```bash
TEST_DATABASE_URL='postgres://postgres:<redacted>@127.0.0.1:5432/tigerfs_test_20261001?sslmode=disable' \
TEST_MOUNT_METHOD=nfs \
go test -list . ./test/integration/...
```

结果：成功列出集成测试，退出码 `0`。这确认测试包可以在当前 Go 工具链和 macOS 环境下编译、初始化并探测本机数据库。

### 3.3 macOS NFS 与数据库冒烟测试

执行了以下测试集合：

```bash
TEST_DATABASE_URL='postgres://postgres:<redacted>@127.0.0.1:5432/tigerfs_test_20261001?sslmode=disable' \
TEST_MOUNT_METHOD=nfs \
go test -count=1 -v -timeout 5m \
  -run '^(TestCRUDFullCycle|TestCRUDPartialUpdates|TestFSOperations_ReadDir_Root|TestFSOperations_ReadDir_Table|TestSynth_HierarchicalWriteFile|TestSynth_HierarchicalReadDir|TestSynth_MkdirAlreadyExists|TestSynth_RenameDirChildrenUnaffected|TestMount_BuildMarkdownViaNFS|TestLargeWrite_64KB|TestImport_OverwriteMode)$' \
  ./test/integration/...
```

结果：成功，退出码 `0`，耗时约 `6.6s`。

覆盖并通过的能力包括：

- PostgreSQL CRUD、部分字段更新和目录/表元数据读取；
- macOS 原生 NFS 挂载；
- 64 KiB 文件写入与读取；
- import overwrite；
- Markdown workspace 通过 NFS 构建；
- 多级目录写入、读取、`mkdir` 重复创建和目录重命名。

### 3.4 内部包单元测试

执行：

```bash
go test -count=1 ./internal/... ./cmd/tigerfs
```

结果：成功，退出码 `0`。通过的包包括 backend、cmd、config、db、format、fs、fs/synth、fuse、logging、mount、nfs 和 util。

### 3.5 全量集成测试

执行：

```bash
TEST_DATABASE_URL='postgres://postgres:<redacted>@127.0.0.1:5432/tigerfs_test_20261001?sslmode=disable' \
TEST_MOUNT_METHOD=nfs \
timeout 12m go test -count=1 -v -timeout 10m ./test/integration/...
```

结果：退出码 `1`，耗时约 `146s`。基础数据库、数据映射、NFS 读写、DDL、导入导出、大文件和格式处理等大量测试通过；失败主要集中在需要历史能力的测试，错误为：

```text
history requires TimescaleDB extension
install TimescaleDB or use a TimescaleDB-enabled PostgreSQL image
```

另有 TimescaleDB hypertable 相关测试因扩展不可用而跳过或无法满足其预期数据集。该结果是测试环境能力边界，不应误判为 macOS NFS 或 Go 构建失败。

### 3.6 直接拉取 TimescaleDB 镜像

为了完全覆盖仓库测试配置，曾通过代理尝试：

```bash
docker pull timescale/timescaledb-ha:pg18
```

结果：Docker daemon 返回：

```text
Error response from daemon: Get "https://registry-1.docker.io/v2/": EOF
```

这表明当前 shell 中的代理环境变量已能帮助 Go 模块下载，但 Docker/OrbStack daemon 的镜像拉取链路没有继承该 shell 代理配置。要运行完整 TimescaleDB 测试，需要另外为 Docker/OrbStack daemon 配置 registry 代理，或预先把镜像导入本机；本次没有修改 Docker daemon 配置。

### 3.7 国内镜像拉取、重标记与全量复测

参考本机其他项目的 `scripts/docker-build-local.sh`，采用“从国内镜像源拉取，再重标记为官方镜像名”的方式。关键点是：镜像源地址只用于 `docker pull`，而 Compose、Testcontainers 和测试连接仍使用仓库原本的官方镜像标签。

本次使用的镜像源是 `docker.m.daocloud.io`：

```bash
MIRROR_BASE_URL=docker.m.daocloud.io
SOURCE_IMAGE=timescale/timescaledb-ha:pg18

docker pull "${MIRROR_BASE_URL}/${SOURCE_IMAGE}"
docker tag "${MIRROR_BASE_URL}/${SOURCE_IMAGE}" "${SOURCE_IMAGE}"
```

这里必须保留完整的命名空间 `timescale/timescaledb-ha:pg18`。与参考脚本中主要处理 Docker Hub 官方短名的场景不同，TimescaleDB 是带组织名的镜像，不能只拼接 `timescaledb-ha:pg18`。

拉取和重标记结果：

```text
source: docker.m.daocloud.io/timescale/timescaledb-ha:pg18
target: timescale/timescaledb-ha:pg18
platform: linux/arm64
PostgreSQL: 18.6
TimescaleDB extension: available
uuidv7(): available
```

随后在隔离端口 `127.0.0.1:55432` 启动容器，并运行：

```bash
TEST_DATABASE_URL='postgres://testuser:<redacted>@127.0.0.1:55432/tigerfs_test?sslmode=disable' \
TEST_MOUNT_METHOD=nfs \
go test -count=1 -timeout 12m ./test/integration/...
```

结果：**成功，退出码 `0`，耗时约 `454s`**。这次完整集成测试覆盖了仓库中的 history、log、savepoint、undo、hypertable、NFS、DDL、导入导出和大文件路径，验证了国内镜像重标记后的 TimescaleDB 环境可以满足 TigerFS 测试要求。

测试结束后已移除临时容器 `tigerfs-timescale-test-20261001`，保留本地镜像及两个标签，便于后续用同样的 `docker run` 命令复测；用户原有的 `postgresql` 容器未作改动。

### 3.8 Apache/Community edition 行为探针

为核实“TigerFS 是否只需要 TimescaleDB Apache 部分”的边界，使用同一 `timescale/timescaledb-ha:pg18` 镜像分别探测默认 Community 模式（`timescaledb.license = timescale`）和 Apache 模式（`timescaledb.license = apache`）。探针表复用了 TigerFS 当前生成的核心 DDL 形状：

```sql
CREATE TABLE tigerfs_probe (
    id BIGINT,
    t TIMESTAMPTZ NOT NULL,
    payload TEXT
) WITH (
    tsdb.hypertable,
    tsdb.partition_column = 't',
    tsdb.chunk_interval = '7 days',
    tsdb.segmentby = 'id',
    tsdb.orderby = 't DESC'
);
```

结果：

| 模式 | 建表 | `timescaledb_information.hypertables.compression_enabled` |
| --- | --- | ---: |
| Community/默认 `timescale` | 成功 | `true` |
| Apache `apache` | 成功 | `false` |

Apache 模式仍支持基础 hypertable 建表，因此不能把“同一 SQL 可以执行”误读成“得到与 Community 相同的 columnstore/compression 能力”。这与 Timescale 官方 edition matrix 的分层一致：hypertable/chunk 基础能力属于 Apache 2 Edition，Hypercore/columnstore 等能力属于 Community Edition。该探针只验证数据库能力边界，不修改 TigerFS 源码，也不代表一次完整 Apache-mode TigerFS history 集成测试。

当前 TigerFS 的前置检查只验证 `timescaledb` 扩展存在，不验证 `timescaledb.license`；因此如果部署者显式启用 Apache 模式，TigerFS 可能不会在入口处报错，而是继续生成没有 Community columnstore/compression 效果的 history/log 表。这是当前实现的部署风险，也是后续应补充的 edition preflight 或能力探测点。

## 4. 结论

1. **构建通过。** 当前 macOS 原生 Go `1.25.14 darwin/arm64` 可以成功构建 TigerFS 全部包。
2. **macOS 运行路径通过。** 使用 Docker PostgreSQL 加 macOS 原生 NFS，TigerFS 的 CRUD、目录、导入导出、大文件和部分 File-first workspace 集成测试可以运行。
3. **内部单元测试通过。** `go test ./internal/... ./cmd/tigerfs` 成功。
4. **vanilla PostgreSQL 16 的全量测试会暴露真实前置条件。** 它没有 TimescaleDB，且缺少 PostgreSQL 18 提供的 `uuidv7()`；本次临时函数只用于诊断，不应作为生产兼容方案。
5. **国内镜像重标记方案可行。** 从 `docker.m.daocloud.io/timescale/timescaledb-ha:pg18` 拉取并重标记为 `timescale/timescaledb-ha:pg18` 后，完整集成测试成功通过。
6. **可复用做法是镜像级绕过，而不是修改仓库测试配置。** 保持 Compose/Testcontainers 中的官方镜像名不变，先在本机准备同名本地标签即可。
7. **当前 history 的 edition 边界已实测。** TigerFS 的 `tsdb.segmentby/orderby` DDL 在 Apache 模式可以建 hypertable，但 `compression_enabled` 为 `false`；完整测试使用的默认 Community 模式为 `true`。因此当前 history 不能宣称已经是 Apache-only。

## 5. 可复用命令模板

```bash
cd /Users/yangjiefeng/Documents/wubuku/tigerfs

# 只为下载资源设置代理
export https_proxy=http://127.0.0.1:9981
export http_proxy=http://127.0.0.1:9981
export all_proxy=socks5://127.0.0.1:9981
export GOPROXY=https://proxy.golang.org,direct

# 本机 Docker PostgreSQL
export TEST_DATABASE_URL='postgres://postgres:<password>@127.0.0.1:5432/tigerfs_test_20261001?sslmode=disable'
export TEST_MOUNT_METHOD=nfs

# 中国境内无法直连 Docker Hub 时，先从国内镜像拉取并重标记
MIRROR_BASE_URL=docker.m.daocloud.io
SOURCE_IMAGE=timescale/timescaledb-ha:pg18
docker pull "${MIRROR_BASE_URL}/${SOURCE_IMAGE}"
docker tag "${MIRROR_BASE_URL}/${SOURCE_IMAGE}" "${SOURCE_IMAGE}"

go mod download
go build ./...
go test -count=1 ./internal/... ./cmd/tigerfs
go test -count=1 -v -timeout 10m ./test/integration/...
```
