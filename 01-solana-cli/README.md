# Solana CLI 工具使用指南

## 目录

1. [环境准备](#1-环境准备)
2. [从源码编译安装](#2-从源码编译安装)
3. [验证安装](#3-验证安装)
4. [基础配置](#4-基础配置)
5. [密钥对管理](#5-密钥对管理)
6. [网络操作](#6-网络操作)
7. [账户操作](#7-账户操作)
8. [转账操作](#8-转账操作)
9. [程序部署](#9-程序部署)
10. [常用命令速查](#10-常用命令速查)

---

## 1. 环境准备

### 系统要求

- 操作系统：Linux / macOS / Windows (WSL2)
- 内存：建议 8GB 以上
- 磁盘：至少 10GB 可用空间

### 安装依赖

**macOS：**

```bash
# 安装 Xcode Command Line Tools
xcode-select --install

# 安装 Homebrew（如未安装）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装必要依赖
brew install pkg-config openssl protobuf
```

**Ubuntu / Debian：**

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential \
  pkg-config \
  libudev-dev \
  llvm \
  libclang-dev \
  protobuf-compiler \
  libssl-dev
```

### 安装 Rust

Solana 是用 Rust 编写的，编译前需要安装 Rust 工具链：

```bash
# 安装 rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 加载环境变量
source "$HOME/.cargo/env"

# 验证安装
rustc --version
cargo --version
```

---

## 2. 从源码编译安装

### 2.1 克隆源码仓库

```bash
# 克隆 Agave 仓库（Solana 的官方社区维护分支，完整仓库较大，约 2GB+）
git clone https://github.com/anza-xyz/agave.git
cd agave
```

### 2.2 切换到稳定版本

```bash
# 切换到指定版本（以 v4.1 为例）
git checkout v4.1
```

### 2.3 编译

```bash
# 编译所有 CLI 工具（耗时较长，视机器性能约 10~30 分钟）
cargo build --release

# 也可以只编译需要的工具，例如只编译 solana CLI：
cargo build --release --bin solana
```

> **提示：** 首次编译时 Rust 会下载所有依赖包，需要保持网络畅通。

编译产物位于 `target/release/` 目录下，主要包含：

| 可执行文件 | 说明 |
|---|---|
| `solana` | 主 CLI 工具 |
| `solana-keygen` | 密钥对生成工具 |
| `solana-validator` | 验证节点程序 |
| `solana-test-validator` | 本地测试验证节点 |
| `solana-bench-tps` | 性能测试工具 |

### 2.4 安装到系统路径

```bash
# 方式一：复制到系统路径
sudo cp target/release/solana* /usr/local/bin/

# 方式二：将 target/release 添加到 PATH
echo 'export PATH="$HOME/solana/target/release:$PATH"' >> ~/.bashrc
source ~/.bashrc

# macOS 用户修改 ~/.zshrc
echo 'export PATH="$HOME/solana/target/release:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 2.5 （可选）使用官方安装脚本

如果不需要修改源码，可以使用官方脚本快速安装预编译版本：

```bash
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
```

安装完成后将提示你将 `~/.local/share/solana/install/active_release/bin` 添加到 `PATH`。

---

## 3. 验证安装

```bash
# 查看版本信息
solana --version
# 输出示例：solana-cli 2.1.0 (src:00000000; feat:1416569292, client:Agave)

# 查看帮助
solana --help
```

---

## 4. 基础配置

Solana CLI 的配置文件默认位于 `~/.config/solana/cli/config.yml`。

### 4.1 查看当前配置

```bash
solana config get
```

输出示例：

```
Config File: /Users/user/.config/solana/cli/config.yml
RPC URL: https://api.mainnet-beta.solana.com
WebSocket URL: wss://api.mainnet-beta.solana.com/ (computed)
Keypair Path: /Users/user/.config/solana/id.json
Commitment: confirmed
```

### 4.2 切换网络（Cluster）

Solana 有三个主要公共网络：

```bash
# 主网
solana config set --url mainnet-beta
# 或
solana config set --url https://api.mainnet-beta.solana.com

# 开发网（有水龙头，可领取测试 SOL）
solana config set --url devnet
# 或
solana config set --url https://api.devnet.solana.com

# 测试网
solana config set --url testnet
# 或
solana config set --url https://api.testnet.solana.com

# 本地测试节点
solana config set --url localhost
# 或
solana config set --url http://127.0.0.1:8899
```

### 4.3 设置默认密钥对路径

```bash
solana config set --keypair ~/.config/solana/id.json
```

---

## 5. 密钥对管理

### 5.1 生成新密钥对

```bash
# 生成默认密钥对（保存到 ~/.config/solana/id.json）
solana-keygen new

# 指定输出路径
solana-keygen new --outfile ~/my-keypair.json

# 生成时不设置密码（不推荐用于主网）
solana-keygen new --no-bip39-passphrase
```

### 5.2 查看公钥地址

```bash
# 查看默认密钥对的公钥
solana-keygen pubkey

# 查看指定密钥对文件的公钥
solana-keygen pubkey ~/my-keypair.json
```

### 5.3 验证密钥对

```bash
# 验证密钥对文件是否有效
solana-keygen verify <PUBLIC_KEY> ~/my-keypair.json
```

### 5.4 生成靓号地址（Vanity Address）

```bash
# 生成以 "abc" 开头的地址（耗时视前缀长度而定）
solana-keygen grind --starts-with abc:1
```

---

## 6. 网络操作

### 6.1 查看集群信息

```bash
# 查看当前连接的集群版本
solana cluster-version

# 查看集群节点信息
solana gossip

# 查看集群的 epoch 信息
solana epoch-info
```

### 6.2 查看区块信息

```bash
# 查看最新区块高度
solana block-height

# 查看指定 slot 的区块信息
solana block <SLOT_NUMBER>
```

### 6.3 查看交易信息

```bash
# 查看交易详情
solana confirm <TRANSACTION_SIGNATURE>

# 查看交易详情（详细模式）
solana confirm -v <TRANSACTION_SIGNATURE>
```

---

## 7. 账户操作

### 7.1 查看账户余额

```bash
# 查看默认账户余额
solana balance

# 查看指定账户余额
solana balance <PUBLIC_KEY>

# 查看余额（以 lamports 为单位，1 SOL = 1,000,000,000 lamports）
solana balance --lamports
```

### 7.2 查看账户详情

```bash
# 查看账户详细信息
solana account <PUBLIC_KEY>

# 以 JSON 格式输出
solana account <PUBLIC_KEY> --output json
```

### 7.3 领取测试 SOL（仅限 devnet/testnet）

```bash
# 领取 1 SOL（devnet）
solana airdrop 1

# 向指定地址领取
solana airdrop 2 <PUBLIC_KEY>
```

---

## 8. 转账操作

```bash
# 向指定地址转账 0.1 SOL
solana transfer <RECIPIENT_PUBLIC_KEY> 0.1

# 转账并指定密钥对
solana transfer <RECIPIENT_PUBLIC_KEY> 0.1 --keypair ~/my-keypair.json

# 转账全部余额（扣除手续费后）
solana transfer <RECIPIENT_PUBLIC_KEY> ALL

# 模拟转账（不实际发送，用于估算手续费）
solana transfer <RECIPIENT_PUBLIC_KEY> 0.1 --simulate
```

---

## 9. 程序部署

### 9.1 部署 BPF 程序

```bash
# 部署编译好的程序（.so 文件）
solana program deploy /path/to/program.so

# 部署并指定程序 ID（需要对应的密钥对文件）
solana program deploy /path/to/program.so --program-id /path/to/program-keypair.json

# 使用缓冲账户部署（适合大程序，支持断点续传）
solana program deploy /path/to/program.so --use-rpc
```

### 9.2 查看已部署程序

```bash
# 查看程序账户信息
solana program show <PROGRAM_ID>

# 列出指定升级权限的所有程序
solana program show --programs --upgrade-authority <AUTHORITY_PUBLIC_KEY>
```

### 9.3 升级程序

```bash
# 升级已部署的程序
solana program deploy /path/to/new_program.so --program-id <PROGRAM_ID>
```

### 9.4 关闭程序（回收租金）

```bash
# 关闭程序并回收 SOL
solana program close <PROGRAM_ID>
```

---

## 10. 常用命令速查

| 命令 | 说明 |
|---|---|
| `solana config get` | 查看当前配置 |
| `solana config set --url <CLUSTER>` | 切换网络 |
| `solana balance` | 查看余额 |
| `solana airdrop <AMOUNT>` | 领取测试 SOL |
| `solana transfer <TO> <AMOUNT>` | 转账 |
| `solana block-height` | 查看区块高度 |
| `solana epoch-info` | 查看 epoch 信息 |
| `solana account <PUBKEY>` | 查看账户信息 |
| `solana confirm <SIG>` | 查看交易确认状态 |
| `solana program deploy <FILE>` | 部署程序 |
| `solana program show <ID>` | 查看程序信息 |
| `solana-keygen new` | 生成新密钥对 |
| `solana-keygen pubkey` | 查看公钥 |
| `solana logs` | 实时查看链上日志 |
| `solana-test-validator` | 启动本地测试节点 |

---

## 参考资源

- [Solana 官方文档](https://docs.solana.com)
- [Solana CLI 参考手册](https://docs.solana.com/cli)
- [Agave 源码仓库](https://github.com/anza-xyz/agave)
- [Solana Cookbook](https://solanacookbook.com)
