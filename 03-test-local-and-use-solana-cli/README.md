# 本地节点实战：solana-test-validator + CLI 全命令演练

> 本文档面向 Solana 开发者，以**本地单节点**为实验环境，系统演练 `solana` CLI 的各类命令。
> 所有操作均在 `localhost` 网络上完成，零成本、零风险。

---

## 目录

1. [环境准备](#1-环境准备)
2. [启动本地节点](#2-启动本地节点)
3. [配置 CLI 连接本地节点](#3-配置-cli-连接本地节点)
4. [密钥对管理](#4-密钥对管理)
5. [账户与余额操作](#5-账户与余额操作)
6. [转账操作](#6-转账操作)
7. [查询链上信息](#7-查询链上信息)
8. [部署与管理程序](#8-部署与管理程序)
9. [质押操作（Staking）](#9-质押操作staking)
10. [Nonce 账户（离线签名）](#10-nonce-账户离线签名)
11. [日志与调试](#11-日志与调试)
12. [常见问题排查](#12-常见问题排查)

---

## 1. 环境准备

### 确认工具已安装

```bash
solana --version
# solana-cli 2.x.x 或 agave-cli 3.x.x

solana-keygen --version

solana-test-validator --version
```

如未安装，请先参考 [01-solana-cli](../01-solana-cli/README.md) 或 [02-agave](../02-agave/README.md) 完成安装。

### 目录结构建议

在开始前，建议创建一个专用工作目录：

```bash
mkdir -p ~/solana-dev/keys
cd ~/solana-dev
```

---

## 2. 启动本地节点

### 2.1 最简启动

```bash
solana-test-validator
```

启动成功后会看到类似输出：

```
Ledger location: test-ledger
Log: test-ledger/validator.log
⠙ Initializing...
Identity: 6Ds7...（节点身份公钥）
Genesis Hash: 9K3B...
Version: 2.1.0
Shred Version: 12345
Gossip Address: 127.0.0.1:1024
TPU Address: 127.0.0.1:1027
JSON RPC URL: http://127.0.0.1:8899
WebSocket PubSub URL: ws://127.0.0.1:8900
⠹ 00:00:09 | Processed Slot: 102 | Confirmed Slot: 102 | Finalized Slot: 71
```

### 2.2 常用启动参数

```bash
solana-test-validator \
    --reset \                            # 清空上次的账本，重新开始
    --ledger /tmp/test-ledger \          # 指定账本目录
    --rpc-port 8899 \                    # RPC 端口（默认 8899）
    --faucet-port 9900 \                 # Faucet 端口（默认 9900）
    --log -                              # 日志直接输出到 stdout（便于调试）
```

### 2.3 预加载已有程序（克隆自主网）

开发时常需要调用 SPL Token、Metaplex 等已部署程序，可以从主网克隆：

```bash
solana-test-validator \
    --reset \
    --url mainnet-beta \
    --clone TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA \   # SPL Token
    --clone metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s     # Metaplex Token Metadata
```

### 2.4 预加载自定义程序

```bash
solana-test-validator \
    --reset \
    --bpf-program <PROGRAM_ID> target/deploy/my_program.so
```

### 2.5 后台运行

```bash
# 后台启动，日志写入文件
nohup solana-test-validator --reset --log /tmp/test-validator.log &
echo "PID: $!"

# 停止节点
kill $(pgrep solana-test-validator)
```

---

## 3. 配置 CLI 连接本地节点

**每次切换网络后都需要重新配置，这是开发中最容易忘记的步骤。**

```bash
# 切换到本地节点
solana config set --url localhost
# 等价写法
solana config set --url http://127.0.0.1:8899

# 查看当前配置，确认切换成功
solana config get
```

输出应包含：

```
RPC URL: http://127.0.0.1:8899
WebSocket URL: ws://127.0.0.1:8900/ (computed)
```

---

## 4. 密钥对管理

### 4.1 生成默认身份密钥对

```bash
# 生成并写入默认路径 ~/.config/solana/id.json
solana-keygen new --outfile ~/.config/solana/id.json

# 如果已有密钥对，加 --force 覆盖
solana-keygen new --outfile ~/.config/solana/id.json --force

# 生成时不设置 BIP39 助记词密码（本地开发可接受）
solana-keygen new --no-bip39-passphrase --outfile ~/.config/solana/id.json
```

### 4.2 生成其他用途密钥对

```bash
# 生成测试用账户 Alice
solana-keygen new --no-bip39-passphrase --outfile ~/solana-dev/keys/alice.json

# 生成测试用账户 Bob
solana-keygen new --no-bip39-passphrase --outfile ~/solana-dev/keys/bob.json
```

### 4.3 查看公钥

```bash
# 查看默认密钥对的公钥
solana-keygen pubkey
# 或
solana address

# 查看指定文件的公钥
solana-keygen pubkey ~/solana-dev/keys/alice.json

# 设置变量方便后续使用
ALICE=$(solana-keygen pubkey ~/solana-dev/keys/alice.json)
BOB=$(solana-keygen pubkey ~/solana-dev/keys/bob.json)
echo "Alice: $ALICE"
echo "Bob:   $BOB"
```

### 4.4 验证密钥对有效性

```bash
solana-keygen verify $ALICE ~/solana-dev/keys/alice.json
# 输出：Verification for public key: Alice... Success
```

### 4.5 生成靓号地址（Vanity Address）

```bash
# 生成以 "Sol" 开头的地址（前缀越长越耗时）
solana-keygen grind --starts-with Sol:1

# 生成以特定字符结尾
solana-keygen grind --ends-with abc:1

# 多线程加速
solana-keygen grind --starts-with dev:1 --num-threads 8
```

---

## 5. 账户与余额操作

### 5.1 领取测试 SOL（Airdrop）

```bash
# 向默认账户领取 10 SOL
solana airdrop 10

# 向指定地址领取
solana airdrop 5 $ALICE
solana airdrop 5 $BOB

# 验证余额
solana balance
solana balance $ALICE
solana balance $BOB
```

> **注意：** 本地节点 Airdrop 无上限，devnet 每次最多 2 SOL。

### 5.2 以 lamports 单位查询

```bash
# 1 SOL = 1,000,000,000 lamports
solana balance --lamports
solana balance $ALICE --lamports
```

### 5.3 查看账户详情

```bash
# 查看账户完整信息（owner、lamports、data、executable 等）
solana account $ALICE

# JSON 格式输出（便于脚本处理）
solana account $ALICE --output json

# 查看账户数据（十六进制）
solana account $ALICE --output json-compact
```

输出字段说明：

| 字段 | 说明 |
|---|---|
| `lamports` | 账户余额（单位 lamport） |
| `data` | 账户存储的数据（普通钱包为空） |
| `owner` | 该账户归属的程序（普通钱包归属 System Program） |
| `executable` | 是否为可执行程序账户 |
| `rentEpoch` | 下次收取租金的 epoch |

---

## 6. 转账操作

### 6.1 基础转账

```bash
# 从默认账户向 Alice 转 1 SOL
solana transfer $ALICE 1

# 使用指定密钥对作为付款方
solana transfer $BOB 0.5 --keypair ~/solana-dev/keys/alice.json

# 查看转账后余额
solana balance $ALICE
solana balance $BOB
```

### 6.2 转出全部余额

```bash
# ALL 会自动扣除手续费后转出剩余全部
solana transfer $BOB ALL --keypair ~/solana-dev/keys/alice.json
```

### 6. 查看转账交易详情

```bash
# 转账时加 --verbose 可直接看到签名
solana transfer $ALICE 0.1 --verbose

# 用签名查询交易详情
solana confirm <TRANSACTION_SIGNATURE>
solana confirm <TRANSACTION_SIGNATURE> --verbose
```

---

## 7. 查询链上信息

### 7.1 集群与节点信息

```bash
# 查看连接集群的版本
solana cluster-version

# 查看当前 epoch 信息
solana epoch-info

# 查看 gossip 中的所有节点
solana gossip

# 查看当前 validators（本地节点只有一个）
solana validators

# 查看供应量信息
solana supply
solana supply --print-accounts  # 详细列出大户账户
```

### 7.2 区块与 Slot 信息

```bash
# 查看最新区块高度
solana block-height

# 查看当前 slot
solana slot

# 查看最新 confirmed slot
solana slot --commitment confirmed

# 查看指定 slot 的区块详情
solana block <SLOT_NUMBER>

# 查看最新区块时间
solana block-time
solana block-time <SLOT_NUMBER>
```

### 7.3 交易查询

```bash
# 查看账户最近的交易记录
solana transaction-history $ALICE

# 限制返回条数
solana transaction-history $ALICE --limit 10

# 查看某笔交易详情
solana confirm <SIGNATURE> --verbose

# 查看交易状态（不含详情，速度更快）
solana confirm <SIGNATURE>
```

### 7.4 Leader Schedule

```bash
# 查看当前 epoch 的 Leader 排期（本地节点只有自己）
solana leader-schedule

# 查看指定 epoch 的排期
solana leader-schedule --epoch <EPOCH_NUMBER>
```

---

## 综合实战脚本

以下脚本整合了本文档全部操作，可直接运行：

```bash
#!/usr/bin/env bash
set -e

echo "=== 1. 启动本地节点（后台）==="
solana-test-validator --reset --ledger /tmp/dev-ledger --log /tmp/dev-validator.log &
VALIDATOR_PID=$!
sleep 5  # 等待节点就绪

echo "=== 2. 配置 CLI ==="
solana config set --url localhost

echo "=== 3. 初始化账户 ==="
solana-keygen new --no-bip39-passphrase --outfile /tmp/alice.json --force
solana-keygen new --no-bip39-passphrase --outfile /tmp/bob.json --force
ALICE=$(solana-keygen pubkey /tmp/alice.json)
BOB=$(solana-keygen pubkey /tmp/bob.json)

echo "=== 4. 领取测试 SOL ==="
solana airdrop 10 $ALICE
solana airdrop 10 $BOB

echo "=== 5. 查看余额 ==="
echo "Alice: $(solana balance $ALICE)"
echo "Bob:   $(solana balance $BOB)"

echo "=== 6. Alice → Bob 转账 1 SOL ==="
solana transfer $BOB 1 --keypair /tmp/alice.json

echo "=== 7. 转账后余额 ==="
echo "Alice: $(solana balance $ALICE)"
echo "Bob:   $(solana balance $BOB)"

echo "=== 8. 查看链上信息 ==="
solana block-height
solana epoch-info

echo "=== 完成！停止节点 ==="
kill $VALIDATOR_PID
```

---