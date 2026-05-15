# Solana 区块链基础

## 概述

Solana 是一条高性能的 Layer 1 区块链，由 Anatoly Yakovenko 于 2017 年创立，主网于 2020 年 3 月上线。其核心设计目标是**在不牺牲去中心化的前提下实现高吞吐量和低延迟**。

## 核心特性

| 特性 | 说明 |
|------|------|
| **高吞吐量** | 理论峰值 65,000+ TPS，实际运行通常数千 TPS |
| **低延迟** | 出块时间约 400ms（以太坊约 12s） |
| **低费用** | 普通交易费用约 $0.00001 |
| **无内存池** | 交易被验证者连续处理，无需排队等待 |

## 关键技术

### Proof of History (PoH) — 历史证明

PoH 是 Solana 最核心的创新。它是一种**加密时钟**，通过 VDF（可验证延迟函数）在交易前就为每个事件打上时间戳：

- 验证者运行 SHA256 哈希链，每个输出作为下一个输入
- 整个过程**不可并行加速**，任何人都能验证时间序列的正确性
- 解决了分布式系统中就事件顺序达成共识的难题

```
SHA256(n) → SHA256(SHA256(n)) → SHA256(SHA256(SHA256(n))) → ...
```

### Tower BFT — 共识机制

基于 PoH 时钟优化的 PBFT（实用拜占庭容错）变体：

- 利用 PoH 作为全局时钟源，减少消息开销
- 验证者无需持续通信即可就状态达成共识
- 支持静态回滚和确定性最终性

### Turbine — 区块传播协议

受 BitTorrent 启发，将区块拆分成小数据包在网络中广播：

- 每个数据包最大 64KB
- 使用喷泉码（erasure codes）保证数据完整性
- 大幅降低验证者的带宽压力

### Gulf Stream — 交易转发协议

抛弃传统"内存池"模型，将交易提前推送给下一批验证者：

- 客户端直接将交易发送给即将出块的验证者
- 减少交易确认时间
- 消除全网广播交易的开销

### Sealevel — 并行智能合约引擎

支持并行处理不相冲突的交易：

- 交易声明了它们将读/写的账户
- 不冲突的交易可以**并行执行**
- 类似操作系统中的乐观并发控制

### Pipeline — 交易处理管道

将交易处理分为多个阶段，类似 CPU 指令流水线：

| 阶段 | 职责 |
|------|------|
| Fetch | 获取交易数据 |
| SigVerify | 验证签名 |
| Banking | 处理账户变动 |
| Write | 写入状态 |

### Cloudbreak — 账户数据库

为并行读写设计的状态存储架构：

- 基于内存映射文件
- 分散在不同的 SSD 上以减少争用
- 支持同步的读写操作

## 核心概念

### 账户模型

Solana 使用**账户（Account）**模型，类似以太坊但更底层：

```
Solana 账户 = 以太坊地址 + 以太坊存储槽
```

- 每个账户有一个**所有者程序（Owner Program）**
- 只有所有者可以修改账户数据
- 账户需要**租金（Rent）**来维持存在（可变为 rent-exempt）

### 程序（Programs）

Solana 的智能合约称为**程序**，是**无状态**的：

- 程序代码和程序状态分离
- 程序代码存储在可执行账户中
- 状态数据存储在其他账户中
- 主要用 **Rust** 编写，编译为 BPF（Berkeley Packet Filter）字节码

### 交易（Transactions）

```rust
// 交易结构示意
Transaction {
    signatures: Vec<Signature>,  // 签名列表
    message: Message {
        header: MessageHeader,
        accounts: Vec<AccountMeta>,  // 涉及的账户
        instructions: Vec<Instruction>,  // 指令列表
    },
}
```

- 一个交易可以包含多个**指令**
- 每个指令调用特定的程序
- 交易声明所有将读/写的账户（支持并行执行）

### 指令（Instructions）

```rust
Instruction {
    program_id: Pubkey,  // 目标程序
    accounts: Vec<AccountMeta>,  // 涉及的账户
    data: Vec<u8>,  // 序列化的参数数据
}
```

### PDA（Program Derived Address）

不是由私钥派生的地址，而是由**程序 ID + 种子**通过确定性算法生成：

```rust
// PDA 示例
let (pda, bump) = Pubkey::find_program_address(
    &[b"seed_prefix", user.key().as_ref()],
    &program_id,
);
```

- PDA 让程序可以"拥有"地址，去掉私钥管理负担
- 常用于合约状态存储、权限控制

## 与以太坊对比

| 维度 | Solana | 以太坊 |
|------|--------|--------|
| 共识 | PoH + PoS (Tower BFT) | PoS (Gasper) |
| TPS | ~3,000-65,000 | ~15-30 (L1) |
| 出块时间 | ~400ms | ~12s |
| 交易费用 | ~$0.00001 | ~$1-$50+ |
| 智能合约语言 | Rust / C / C++ | Solidity / Vyper |
| 合约模型 | 无状态程序 | 状态耦合的合约 |
| 执行模型 | 并行 (Sealevel) | 顺序 (EVM) |
| 地址模型 | Ed25519 | ECDSA (secp256k1) |

## 开发工具链

### CLI 工具

```bash
# 安装 Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# 查看配置
solana config get

# 切换到 devnet
solana config set --url devnet

# 创建钱包
solana-keygen new

# 查询余额
solana balance

# 空投测试币
solana airdrop 1
```

### Anchor 框架

Anchor 是 Solana 上最主流的开发框架，类似以太坊的 Hardhat：

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        let account = &mut ctx.accounts.my_account;
        account.data = data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub my_account: Account<'info, MyAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### SPL Token

Solana 上的代币标准（类似 ERC-20 / ERC-721）：

- `spl-token create-token` — 创建代币
- `spl-token create-account` — 创建代币账户
- `spl-token mint` — 铸造代币

## 网络环境

| 网络 | 用途 |
|------|------|
| **Mainnet Beta** | 主网，真实资产交易 |
| **Devnet** | 开发网，免费空投测试币 |
| **Testnet** | 测试网，测试新功能 |
| **Localnet** | 本地开发网络 |

## 学习资源

- [Solana 官方文档](https://docs.solana.com)
- [Anchor 框架文档](https://www.anchor-lang.com)
- [Solana Cookbook](https://solanacookbook.com)
- [Solana Playground](https://beta.solpg.io) — 在线 IDE
