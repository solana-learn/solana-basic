# Agave —— Solana 验证节点客户端

> 仓库地址：[https://github.com/anza-xyz/agave](https://github.com/anza-xyz/agave)  
> 维护方：[Anza](https://anza.xyz/)  
> 许可证：Apache-2.0

---

## 目录

1. [项目简介](#1-项目简介)
2. [核心架构原理](#2-核心架构原理)
3. [仓库结构概览](#3-仓库结构概览)
4. [环境依赖](#4-环境依赖)
5. [从源码构建](#5-从源码构建)
6. [运行验证节点](#6-运行验证节点)
7. [运行测试](#7-运行测试)
8. [启动本地测试节点](#8-启动本地测试节点)
9. [SBF 程序开发工具链](#9-sbf-程序开发工具链)
10. [Geyser 插件开发](#10-geyser-插件开发)
11. [Feature Gate 机制](#11-feature-gate-机制)
12. [性能基准测试](#12-性能基准测试)
13. [代码覆盖率](#13-代码覆盖率)
14. [发布流程](#14-发布流程)
15. [参考资源](#15-参考资源)

---

## 1. 项目简介

**Agave** 是 Solana 区块链的验证节点（Validator）客户端实现，由 **Anza** 团队维护。它从 [solana-labs/solana](https://github.com/solana-labs/solana) fork 而来，是目前 Solana 网络上使用最广泛的客户端实现之一。

### 核心定位

| 属性 | 说明 |
|---|---|
| 语言 | Rust（占比 97.7%） |
| 定位 | Solana 全节点 / 验证节点客户端 |
| 当前最新版本 | v3.1.14（2025年） |
| Stars | 1.8k |
| Forks | 1k |

### 与 solana-labs/solana 的关系

Agave 在 solana-labs/solana 的基础上持续演进，相比上游已**领先 7000+ 个提交**，是 Solana 生态中最活跃的验证节点实现。

---

## 2. 核心架构原理

理解 Agave 的架构是深度参与开发的前提，以下是几个关键子系统的工作原理。

### 2.1 Proof of History（PoH）

PoH 是 Solana 的核心时间戳机制，本质是一条**连续的 SHA-256 哈希链**：

```
hash[n] = SHA256(hash[n-1] || data)
```

- `poh/` 模块由一个独立线程运行，持续计算哈希序列，相当于一个**可验证的硬件时钟**
- 每个 Entry（条目）包含若干笔交易及其对应的 PoH 哈希，证明这些交易发生在特定时间点
- 验证者无需通信即可独立验证事件的先后顺序，大幅降低共识通信开销

### 2.2 Tower BFT 共识（现有）

Agave 目前使用 **Tower BFT**，它是 PBFT 的 PoH 优化版本：

- 验证节点对 **Slot** 进行投票，投票被编码为交易广播到网络
- 每一层投票都带有**锁仓期（lockout）**，锁仓期随投票深度指数增长（$2^n$ 个 slot）
- 超过 2/3 质押权重的确认形成最终确认（Finality）
- 核心代码位于 `core/src/consensus/` 和 `vote/`

### 2.3 Alpenglow 共识（新一代，开发中）

Alpenglow 是 Anza 正在开发的下一代共识协议，旨在将确认时间从 ~400ms 降低至 **~100ms**：

- 引入 **Votor** 组件（`votor/` 目录）替代 Tower BFT 的投票机制
- 采用两阶段投票（Notarize + Finalize），支持更快速的最终确认
- 使用 **BLS 签名聚合**（`bls-cert-verify/`）压缩证书大小
- 当前通过 Feature Gate 控制，未正式激活主网

### 2.4 Turbine 数据分发

区块数据不直接广播给所有节点，而是使用 **Turbine** 协议进行分层传播：

```
Leader → 根节点层 → 第一层邻居 → 第二层邻居 → ...
```

- 区块被切分为 **Shred**（碎片，`ledger/src/shred/`），每个 Shred 约 1228 字节
- 使用 **Reed-Solomon 纠删码**（erasure coding）保证部分 Shred 丢失时仍可恢复
- Shred 在 `turbine/` 中按 Stake 权重分层转发，减少 Leader 的出口带宽压力

### 2.5 Gulf Stream（交易转发）

Gulf Stream 是 Solana 无内存池（Mempool-less）设计的关键：

- 客户端可预测下一个 Leader，**提前**将交易发送到 TPU（Transaction Processing Unit）
- 当前 Leader 将未处理交易转发给未来 Leader，最多提前 4 个 epoch
- 核心代码在 `core/src/tpu.rs` 和 `banking-stage-ingress-types/`

### 2.6 Banking Stage 流水线

交易从接收到入账经历严格的流水线：

```
sigverify（签名验证）
    ↓
banking stage（账户锁定 → 执行 → 提交）
    ↓
PoH Recorder（写入 PoH 序列）
    ↓
Blockstore（持久化到账本）
```

- `perf/` 提供 SIMD 加速的批量签名验证（Ed25519）
- Banking Stage 使用**账户读写锁**避免同一账户的并发冲突
- SVM（`svm/`）负责实际执行交易中的程序调用

### 2.7 账户模型与 AccountsDB

Solana 的状态存储在**账户（Account）**中，而非合约存储（Storage）：

- 每个账户有 `owner`、`data`、`lamports`、`executable` 等字段
- `accounts-db/` 实现了基于**追加写入（append-only）**的存储引擎，周期性做快照（Snapshot）
- 账户的存储使用 **AccountStorage** 管理分片文件，避免大文件随机写入
- **Rent**（租金）机制要求账户余额覆盖存储成本，否则账户被回收

---

## 3. 仓库结构概览

Agave 是一个 Cargo workspace（Rust monorepo），包含数十个 crate，以下是核心模块说明：

### 核心运行时

| 目录 | 说明 |
|---|---|
| `core/` | 验证节点核心逻辑：区块生产、投票、turbine 数据分发等 |
| `runtime/` | 交易执行运行时，账户状态管理 |
| `svm/` | Solana Virtual Machine（SVM）实现，负责执行 BPF 程序 |
| `program-runtime/` | 智能合约运行时，管理调用栈、计算单元等 |
| `accounts-db/` | 账户数据库，持久化存储所有账户状态 |
| `ledger/` | 账本（Blockstore）管理，存储区块和分片（Shred）数据 |

### 共识与网络

| 目录 | 说明 |
|---|---|
| `gossip/` | Gossip 协议实现，节点间信息传播 |
| `turbine/` | 区块数据分发协议（类 turbine 广播） |
| `poh/` | Proof of History（历史证明）实现 |
| `votor/` | 共识投票逻辑（Alpenglow 共识相关） |
| `streamer/` | QUIC/UDP 数据流处理 |
| `perf/` | 高性能数据包批处理 |

### RPC 与客户端

| 目录 | 说明 |
|---|---|
| `rpc/` | JSON-RPC 服务端实现 |
| `rpc-client/` | RPC 客户端 |
| `pubsub-client/` | WebSocket PubSub 客户端 |
| `banks-client/` | BanksClient，用于测试与程序开发 |
| `tpu-client/` | TPU（Transaction Processing Unit）客户端 |

### CLI 工具

| 目录 | 说明 |
|---|---|
| `cli/` | `solana` 主命令行工具 |
| `keygen/` | `solana-keygen` 密钥对管理工具 |
| `validator/` | `agave-validator` 验证节点启动入口 |
| `ledger-tool/` | `agave-ledger-tool` 账本离线分析工具 |
| `bench-tps/` | `solana-bench-tps` 性能测试工具 |
| `faucet/` | `solana-faucet` 测试水龙头服务 |

### 程序（内置合约）

| 目录 | 说明 |
|---|---|
| `programs/` | 系统内置程序（system、stake、vote、BPF loader 等） |
| `builtins/` | 内置程序注册与管理 |
| `precompiles/` | 预编译程序（ed25519、secp256k1 签名验证等） |

### 测试与开发

| 目录 | 说明 |
|---|---|
| `test-validator/` | `solana-test-validator` 本地单节点测试网 |
| `local-cluster/` | 本地多节点集群测试框架 |
| `program-test/` | 智能合约单元测试框架 |
| `svm-test-harness/` | SVM 独立测试工具 |
| `client-test/` | 客户端集成测试 |

### Geyser 插件

| 目录 | 说明 |
|---|---|
| `geyser-plugin-interface/` | Geyser 插件接口定义（用于实时数据流） |
| `geyser-plugin-manager/` | Geyser 插件加载与管理 |

---

## 4. 环境依赖

### 安装 Rust 工具链

```bash
curl https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env

# 添加 rustfmt 组件
rustup component add rustfmt
```

> Agave 通过根目录的 `rust-toolchain.toml` 文件锁定 Rust 版本（当前为 **1.95.0**），cargo 会自动安装对应版本，无需手动指定。

### Linux 系统依赖

**Ubuntu / Debian：**

```bash
sudo apt-get update
sudo apt-get install -y \
  libssl-dev \
  libudev-dev \
  pkg-config \
  zlib1g-dev \
  llvm \
  clang \
  cmake \
  make \
  libprotobuf-dev \
  protobuf-compiler \
  libclang-dev
```

**Fedora：**

```bash
sudo dnf install \
  openssl-devel \
  systemd-devel \
  pkg-config \
  zlib-devel \
  llvm \
  clang \
  cmake \
  make \
  protobuf-devel \
  protobuf-compiler \
  perl-core \
  libclang-dev
```

**macOS：**

```bash
xcode-select --install
brew install pkg-config openssl protobuf
```

---

## 5. 从源码构建

### 4.1 克隆仓库

```bash
git clone https://github.com/anza-xyz/agave.git
cd agave
```

### 4.2 切换到稳定版本（推荐）

```bash
# 查看所有发布标签
git tag --sort=-v:refname | head -20

# 切换到指定版本
git checkout v3.1.14
```

### 4.3 Debug 构建（快速编译，不适合生产）

```bash
./cargo build
```

产物输出到 `target/debug/`。

### 4.4 Release 构建（推荐用于测试网 / 主网）

```bash
# 构建所有组件
cargo build --release

# 仅构建 CLI 工具
cargo build --release --bin solana
cargo build --release --bin agave-validator
cargo build --release --bin solana-test-validator
```

产物输出到 `target/release/`。

> **注意：** Release 构建耗时较长（首次约 20~40 分钟），请确保网络畅通以下载依赖。

### 4.5 安装到系统路径

```bash
# 复制常用工具到 /usr/local/bin
sudo cp target/release/solana \
        target/release/solana-keygen \
        target/release/agave-validator \
        target/release/solana-test-validator \
        /usr/local/bin/

# 验证
solana --version
agave-validator --version
```

### 4.6 主要可执行文件一览

| 可执行文件 | 说明 |
|---|---|
| `solana` | 主 CLI 工具 |
| `solana-keygen` | 密钥对管理 |
| `agave-validator` | 验证节点主程序 |
| `solana-test-validator` | 本地测试节点 |
| `agave-ledger-tool` | 账本离线分析 |
| `solana-bench-tps` | 交易吞吐量性能测试 |
| `solana-faucet` | 测试水龙头服务 |

---

## 6. 运行验证节点

### 6.1 硬件要求

运行主网验证节点对硬件要求较高：

| 组件 | 最低要求 | 推荐配置 |
|---|---|---|
| CPU | 12 核 / 2.8GHz | AMD EPYC / Intel Xeon，24 核+ |
| RAM | 256 GB | 512 GB |
| 系统盘（OS + 账本） | 500 GB NVMe | 2 TB NVMe（PCIe 4.0） |
| 账户盘（AccountsDB） | 独立 500 GB NVMe | 独立 1 TB NVMe |
| 网络 | 1 Gbps | 10 Gbps，低延迟 |

> 账户盘和账本盘**强烈建议分离**，账户随机写压力极大，会与顺序写账本产生 I/O 竞争。

### 6.2 系统调优

```bash
# 增大 UDP 缓冲区（Solana 大量使用 UDP）
sudo bash -c "cat >> /etc/sysctl.conf << EOF
net.core.rmem_default=134217728
net.core.rmem_max=134217728
net.core.wmem_default=134217728
net.core.wmem_max=134217728
EOF"
sudo sysctl -p

# 关闭透明大页（避免内存延迟抖动）
sudo bash -c "echo madvise > /sys/kernel/mm/transparent_hugepage/enabled"

# 增加文件描述符上限
sudo bash -c "cat >> /etc/security/limits.conf << EOF
* soft nofile 1000000
* hard nofile 1000000
EOF"
```

### 6.3 生成投票账户与身份密钥

```bash
# 生成验证节点身份密钥（validator identity）
solana-keygen new -o ~/validator-keypair.json

# 生成投票账户密钥
solana-keygen new -o ~/vote-account-keypair.json

# 生成提款授权密钥（离线冷存储，重要！）
solana-keygen new -o ~/authorized-withdrawer-keypair.json

# 创建投票账户（需要身份账户有足够 SOL）
solana create-vote-account \
    ~/vote-account-keypair.json \
    ~/validator-keypair.json \
    ~/authorized-withdrawer-keypair.json \
    --commission 10
```

### 6.4 启动验证节点

```bash
agave-validator \
    --identity ~/validator-keypair.json \
    --vote-account ~/vote-account-keypair.json \
    --known-validator 7Np41oeYqPefeNQEHSv1UDhYrehxin3NStELsSKCT4K2 \
    --known-validator GdnSyH3YtwcxFvQrVVJMm1JhTS4QVX7MFsX56uJLUfiZ \
    --only-known-rpc \
    --ledger /mnt/ledger \
    --accounts /mnt/accounts \
    --rpc-port 8899 \
    --dynamic-port-range 8000-8020 \
    --entrypoint entrypoint.mainnet-beta.solana.com:8001 \
    --entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
    --expected-genesis-hash 5eykt4UsFv8P8NJdTREpY1uz1te2NpsNA8jp3Srs5tqM \
    --wal-recovery-mode skip_any_corrupted_record \
    --limit-ledger-size 50000000 \
    --log ~/validator.log
```

**关键参数说明：**

| 参数 | 说明 |
|---|---|
| `--known-validator` | 指定可信节点，防止被恶意快照欺骗 |
| `--only-known-rpc` | 只从可信节点下载快照 |
| `--ledger` | 账本存储路径（建议 NVMe） |
| `--accounts` | 账户数据库路径（建议独立 NVMe） |
| `--limit-ledger-size` | 限制账本保留的 slot 数量，控制磁盘占用 |
| `--wal-recovery-mode` | 异常重启时的账本恢复策略 |

### 6.5 监控节点状态

```bash
# 查看节点追赶进度
solana catchup ~/validator-keypair.json

# 查看节点健康状态（通过 RPC）
curl -s http://localhost:8899/health

# 查看当前 epoch 的 slot 信息
solana epoch-info

# 查看验证节点在当前 epoch 的投票情况
solana vote-account ~/vote-account-keypair.json

# 查看节点实时日志（过滤关键信息）
tail -f ~/validator.log | grep -E "ERROR|WARN|voted|leader"
```

---

## 7. 运行测试

Agave 使用 [cargo-nextest](https://nexte.st/) 作为测试运行器：

```bash
# 安装 nextest（首次需要）
cargo install cargo-nextest

# 运行完整测试套件（CI 配置）
./cargo nextest run --profile ci --cargo-profile ci --config-file .config/nextest.toml

# 运行指定 crate 的测试
cargo test -p solana-runtime

# 运行单个测试
cargo test -p solana-runtime test_name
```

### 常见测试技巧

```bash
# 跳过耗时集成测试，只跑单元测试
cargo nextest run --profile ci -E 'not test(test_full_snapshot)'

# 运行所有 crate 的库测试
cargo test --workspace --lib

# 查看详细输出（nextest 默认隐藏 stdout）
cargo nextest run -p solana-core -- --no-capture

# 限制并发度，防止内存不足
cargo nextest run -p solana-runtime --test-threads 4
```

---

## 8. 启动本地测试节点

`solana-test-validator` 可在本地快速启动一个单节点 Solana 网络，非常适合合约开发与调试。

```bash
# 启动本地测试节点（默认 RPC 8899，Faucet 9900）
solana-test-validator

# 配置 CLI 连接到本地节点
solana config set --url localhost

# 查看节点状态
solana cluster-version
solana block-height
```

### 高级选项

```bash
# 克隆主网账户到本地（用于集成测试，避免在主网上操作）
solana-test-validator \
    --clone <PROGRAM_ID> \
    --url mainnet-beta

# 预加载自定义程序
solana-test-validator \
    --bpf-program <PROGRAM_ID> target/deploy/my_program.so

# 设置初始账户余额（SOL）
solana-test-validator \
    --account <PUBKEY> <ACCOUNT_JSON_FILE>

# 修改区块时间（加速测试，单位毫秒）
solana-test-validator --ticks-per-slot 8

# 指定数据目录（默认 test-ledger/）
solana-test-validator --ledger /tmp/test-ledger

# 重置账本（清空状态重新开始）
solana-test-validator --reset
```

---

## 9. SBF 程序开发工具链

Solana 上的智能合约被编译为 **sBPF**（Solana BPF）字节码，Agave 仓库自带构建工具。

### 9.1 安装 cargo-build-sbf

```bash
# 从 agave 仓库安装（推荐，与节点版本匹配）
cargo install --path ./cargo-build-sbf

# 或使用官方发布版
cargo install cargo-build-sbf
```

### 9.2 编译 Solana 程序

```bash
# 在程序目录下编译（生成 .so 文件）
cargo build-sbf

# 编译并指定目标特性
cargo build-sbf --features my-feature

# 编译后输出到自定义路径
cargo build-sbf -- --manifest-path programs/my_program/Cargo.toml
```

编译产物默认在 `target/deploy/<program_name>.so`。

### 9.3 运行 SBF 程序单元测试

```bash
# 使用 program-test 框架运行测试（不需要启动节点）
cargo test-sbf

# 调试模式（输出更详细日志）
RUST_LOG=solana_program_test=debug cargo test-sbf
```

### 9.4 sBPF 版本演进

| 版本 | 说明 |
|---|---|
| SBPFv0 | 早期版本，基于 rBPF |
| SBPFv1 | 引入更多指令优化 |
| SBPFv2 | 增加动态帧指针等特性 |
| SBPFv3 | 最新版本，进一步优化指令集，当前 CI 使用 |

---

## 10. Geyser 插件开发

**Geyser** 是 Agave 提供的实时数据流插件接口，允许外部程序订阅账户变更、交易、区块等事件，是构建索引器、DEX 数据源的核心机制。

### 10.1 插件接口

Geyser 插件实现 `GeyserPlugin` trait（定义在 `geyser-plugin-interface/`）：

```rust
use agave_geyser_plugin_interface::geyser_plugin_interface::{
    GeyserPlugin, ReplicaAccountInfoVersions, ReplicaTransactionInfoVersions,
    ReplicaBlockInfoVersions, Result as PluginResult,
};

#[derive(Debug)]
pub struct MyPlugin;

impl GeyserPlugin for MyPlugin {
    fn name(&self) -> &'static str {
        "my-plugin"
    }

    fn on_load(&mut self, config_file: &str, is_reload: bool) -> PluginResult<()> {
        // 加载配置、建立数据库连接等初始化操作
        Ok(())
    }

    fn update_account(
        &self,
        account: ReplicaAccountInfoVersions,
        slot: u64,
        is_startup: bool,
    ) -> PluginResult<()> {
        // 处理账户变更事件
        Ok(())
    }

    fn notify_transaction(
        &self,
        transaction: ReplicaTransactionInfoVersions,
        slot: u64,
    ) -> PluginResult<()> {
        // 处理交易事件
        Ok(())
    }
}

// 必须导出此函数，验证节点通过 dlopen 加载插件
#[no_mangle]
pub fn _create_plugin() -> *mut dyn GeyserPlugin {
    Box::into_raw(Box::new(MyPlugin))
}
```

### 10.2 编译为动态库

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib", "rlib"]
```

```bash
cargo build --release
# 产物：target/release/libmy_plugin.so（Linux）
#        target/release/libmy_plugin.dylib（macOS）
```

### 10.3 配置验证节点加载插件

```bash
agave-validator \
    --geyser-plugin-config /path/to/plugin-config.json \
    ...
```

`plugin-config.json` 示例：

```json
{
    "libpath": "/path/to/libmy_plugin.so",
    "my_custom_key": "value"
}
```

---

## 11. Feature Gate 机制

Solana 使用 **Feature Gate** 来安全地在网络上激活新功能，避免硬分叉。

### 11.1 原理

- 每个 Feature 是一个链上账户（Program ID），当账户被激活（`activated_at` 字段被设置）时，对应功能在节点中生效
- Feature 集合定义在 `feature-set/src/lib.rs`，每个 Feature 是一个 `Pubkey` 常量
- Runtime 在每个 epoch 边界检查是否有新 Feature 达到激活门槛（超过 95% 质押支持）

### 11.2 查看 Feature 状态

```bash
# 查看所有 feature 的激活状态
solana feature status

# 查看特定 feature
solana feature status <FEATURE_PUBKEY>

# 在 devnet 激活某个 feature（需要 feature 账户的签名权限）
solana feature activate <FEATURE_KEYPAIR> --cluster devnet
```

### 11.3 在代码中使用 Feature Gate

```rust
// runtime 中判断 feature 是否激活
if invoke_context
    .feature_set
    .is_active(&feature_set::my_new_feature::id())
{
    // 执行新逻辑
} else {
    // 兼容旧逻辑
}
```

---

## 12. 性能基准测试

```bash
# 安装 nightly 工具链（cargo bench 依赖 unstable 特性）
rustup install nightly

# 运行所有基准测试
cargo +nightly bench

# 运行指定 crate 的基准测试
cargo +nightly bench -p solana-runtime

# 运行 TPS 基准测试（针对真实集群）
solana-bench-tps \
    --url http://127.0.0.1:8899 \
    --identity ~/validator-keypair.json \
    --duration 60 \
    --tx-count 5000
```

---

## 13. 代码覆盖率

```bash
# 生成覆盖率报告
./scripts/coverage.sh

# 在浏览器中查看报告
open target/cov/lcov-local/index.html
```

---

## 14. 发布流程

Agave 遵循语义化版本规范，发布流程详见仓库根目录的 [RELEASE.md](https://github.com/anza-xyz/agave/blob/master/RELEASE.md)。

版本号规则：`v<MAJOR>.<MINOR>.<PATCH>`

- **MAJOR**：与 Solana 协议版本对齐
- **MINOR**：功能更新，每个 epoch 约 2 天发布一次小版本
- **PATCH**：热修复

当前发布状态：
- [GitHub Releases 页面](https://github.com/anza-xyz/agave/releases)（共 154 个版本）
- [CI 构建状态](https://buildkite.com/anza/agave-secondary)

---

## 15. 参考资源

| 资源 | 链接 |
|---|---|
| Agave 源码仓库 | https://github.com/anza-xyz/agave |
| Anza 官网 | https://anza.xyz |
| Agave 官方文档 | https://docs.anza.xyz |
| crates.io 发布包 | https://crates.io/crates/agave-validator |
| API 文档 | https://docs.rs/agave-validator |
| Solana 官方文档 | https://docs.solana.com |
| Solana Cookbook | https://solanacookbook.com |
| Geyser 插件示例 | https://github.com/anza-xyz/agave/tree/master/geyser-plugin-interface |
| sBPF 文档 | https://docs.solana.com/developing/on-chain-programs/overview |

