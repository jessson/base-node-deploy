# 不使用 Docker 运行 Base 节点指南

本指南说明如何在不使用 Docker 的情况下运行 Base 节点。

## 前置要求

### 系统要求
- **CPU**: 多核现代 CPU
- **内存**: 32GB (推荐 64GB)
- **存储**: NVMe SSD，足够存储链数据
- **操作系统**: Linux 或 macOS

### 需要安装的软件
1. **Go** (版本 1.24+): 用于编译 op-node
2. **Rust** (版本 1.88+): 用于编译 base-reth-node
3. **Git**: 用于克隆代码仓库
4. **构建工具**:
   - `make` (用于编译 op-node)
   - `cargo` (Rust 包管理器，随 Rust 安装)
   - `just` (构建工具，用于 op-node)
   - `pkg-config`, `libclang-dev`, `build-essential` (Linux)
   - `jq`, `curl` (可选，用于脚本)

## 步骤 1: 编译 op-node

### 1.1 设置环境变量
从 `versions.env` 文件中读取版本信息：

```bash
source versions.env
# 或者手动设置：
export OP_NODE_REPO=https://github.com/ethereum-optimism/optimism.git
export OP_NODE_TAG=op-node/v1.16.2
export OP_NODE_COMMIT=1b8c541060f0d323a7023fbc68fbbc8daf674340
```

### 1.2 克隆并编译 op-node

```bash
# 创建工作目录
mkdir -p ~/base-node
cd ~/base-node

# 克隆代码
git clone $OP_NODE_REPO --branch $OP_NODE_TAG --single-branch op-node
cd op-node

# 验证提交哈希
git switch -c branch-$OP_NODE_TAG
bash -c '[ "$(git rev-parse HEAD)" = "$OP_NODE_COMMIT" ]'

# 安装 just 构建工具
curl -sSfL 'https://just.systems/install.sh' | bash -s -- --to ~/.local/bin
export PATH="$HOME/.local/bin:$PATH"

# 编译 op-node
cd op-node
make VERSION=$OP_NODE_TAG op-node

# 二进制文件将位于: op-node/bin/op-node
```

## 步骤 2: 编译 base-reth-node

### 2.1 设置环境变量

```bash
source versions.env
# 或者手动设置：
export BASE_RETH_NODE_REPO=git@github.com:jessson/base-node.git
export BASE_RETH_NODE_TAG=v0.2.2
export BASE_RETH_NODE_COMMIT=4a580faec0b560eb65d6ee2f620efb7b044a649d
```

### 2.2 安装 Rust 依赖（Linux）

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y git libclang-dev pkg-config curl build-essential
```

### 2.3 克隆并编译 base-reth-node

```bash
cd ~/base-node

# 克隆代码
git clone $BASE_RETH_NODE_REPO base-reth-node
cd base-reth-node
git checkout tags/$BASE_RETH_NODE_TAG

# 验证提交哈希
bash -c '[ "$(git rev-parse HEAD)" = "$BASE_RETH_NODE_COMMIT" ]' || (echo "Commit hash verification failed" && exit 1)

# 编译（使用 maxperf profile 以获得最佳性能）
cargo build --bin base-reth-node --profile maxperf

# 二进制文件将位于: target/maxperf/base-reth-node
```

## 步骤 3: 准备运行环境

### 3.1 创建数据目录

```bash
mkdir -p ~/base-node-data
export DATA_DIR=~/base-node-data
```

### 3.2 创建 JWT secret 文件

```bash
# 生成一个随机的 32 字节 hex 字符串作为 JWT secret
openssl rand -hex 32 > $DATA_DIR/jwt.hex

# 设置环境变量
export OP_NODE_L2_ENGINE_AUTH=$DATA_DIR/jwt.hex
export OP_NODE_L2_ENGINE_AUTH_RAW=$(cat $DATA_DIR/jwt.hex)
```

### 3.3 配置网络环境变量

创建配置文件 `~/.base-node-env`：

```bash
# L1 配置（必需）
export OP_NODE_L1_ETH_RPC=<your-l1-rpc-endpoint>
export OP_NODE_L1_BEACON=<your-l1-beacon-endpoint>
export OP_NODE_L1_BEACON_ARCHIVER=<your-l1-beacon-archiver-endpoint>
export OP_NODE_L1_RPC_KIND=debug_geth  # 或 alchemy, quicknode, infura 等

# 网络配置（主网）
export RETH_CHAIN=base
export OP_NODE_NETWORK=base-mainnet
export RETH_SEQUENCER_HTTP=https://mainnet-sequencer.base.org

# 端口配置（可选，使用默认值）
export RPC_PORT=8545
export WS_PORT=8546
export AUTHRPC_PORT=8551
export METRICS_PORT=6060
export DISCOVERY_PORT=30303
export P2P_PORT=30303

# 数据目录
export RETH_DATA_DIR=$DATA_DIR
export IPC_PATH=$DATA_DIR/reth.ipc
export RETH_LOG_FILE=$DATA_DIR/reth.log

# JWT 配置（已在上面设置）
# export OP_NODE_L2_ENGINE_AUTH=$DATA_DIR/jwt.hex
# export OP_NODE_L2_ENGINE_AUTH_RAW=$(cat $DATA_DIR/jwt.hex)

# op-node 配置
export OP_NODE_L2_ENGINE_RPC=ws://127.0.0.1:8551
export OP_NODE_P2P_ADVERTISE_IP=$(curl -s http://ifconfig.me)  # 自动获取公网 IP
```

## 步骤 4: 运行节点

### 4.1 启动 base-reth-node（执行层）

在一个终端窗口中：

```bash
cd ~/base-node
source ~/.base-node-env

# 确保 JWT secret 文件存在
echo "$OP_NODE_L2_ENGINE_AUTH_RAW" > "$OP_NODE_L2_ENGINE_AUTH"

# 运行 base-reth-node
./base-reth-node/target/maxperf/base-reth-node node \
  -vvv \
  --full \
  --datadir="$RETH_DATA_DIR" \
  --log.file.path="$RETH_LOG_FILE" \
  --log.file.format json \
  --ws \
  --ws.origins="*" \
  --ws.addr=127.0.0.1 \
  --ws.port="$WS_PORT" \
  --ws.api=web3,debug,eth,net,txpool \
  --http \
  --http.corsdomain="*" \
  --http.addr=127.0.0.1 \
  --http.port="$RPC_PORT" \
  --http.api=web3,debug,eth,net,txpool,miner \
  --ipcpath="$IPC_PATH" \
  --authrpc.addr=0.0.0.0 \
  --authrpc.port="$AUTHRPC_PORT" \
  --authrpc.jwtsecret="$OP_NODE_L2_ENGINE_AUTH" \
  --metrics=0.0.0.0:"$METRICS_PORT" \
  --max-outbound-peers=200 \
  --chain "$RETH_CHAIN" \
  --rollup.sequencer-http="$RETH_SEQUENCER_HTTP" \
  --rollup.disable-tx-pool-gossip \
  --discovery.port="$DISCOVERY_PORT" \
  --port="$P2P_PORT"
```

### 4.2 等待执行层就绪

等待 base-reth-node 启动完成。你可以通过检查日志或尝试连接来确认：

```bash
# 检查 authrpc 是否就绪（应该返回 401）
curl -s -w '%{http_code}' -o /dev/null http://127.0.0.1:8551
```

### 4.3 启动 op-node（共识层）

在另一个终端窗口中：

```bash
cd ~/base-node
source ~/.base-node-env

# 确保 JWT secret 文件存在
echo "$OP_NODE_L2_ENGINE_AUTH_RAW" > "$OP_NODE_L2_ENGINE_AUTH"

# 等待执行层就绪
until [ "$(curl -s --max-time 10 --connect-timeout 5 -w '%{http_code}' -o /dev/null http://127.0.0.1:8551)" -eq 401 ]; do
  echo "等待执行层就绪..."
  sleep 5
done

# 获取公网 IP（用于 P2P 广告）
export OP_NODE_P2P_ADVERTISE_IP=$(curl -s http://ifconfig.me || curl -s http://api.ipify.org || curl -s http://ipecho.net/plain || curl -s http://v4.ident.me)

# 运行 op-node
./op-node/op-node/bin/op-node
```

## 步骤 5: 使用 systemd 管理服务（可选）

为了在生产环境中更好地管理服务，可以使用 systemd。

### 5.1 创建 base-reth-node.service

创建文件 `/etc/systemd/system/base-reth-node.service`：

```ini
[Unit]
Description=Base Reth Node (Execution Layer)
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/your-user/base-node
EnvironmentFile=/home/your-user/.base-node-env
ExecStart=/home/your-user/base-node/base-reth-node/target/maxperf/base-reth-node node \
  -vvv \
  --full \
  --datadir=${RETH_DATA_DIR} \
  --log.file.path=${RETH_LOG_FILE} \
  --log.file.format json \
  --ws \
  --ws.origins="*" \
  --ws.addr=127.0.0.1 \
  --ws.port=${WS_PORT} \
  --ws.api=web3,debug,eth,net,txpool \
  --http \
  --http.corsdomain="*" \
  --http.addr=127.0.0.1 \
  --http.port=${RPC_PORT} \
  --http.api=web3,debug,eth,net,txpool,miner \
  --ipcpath=${IPC_PATH} \
  --authrpc.addr=0.0.0.0 \
  --authrpc.port=${AUTHRPC_PORT} \
  --authrpc.jwtsecret=${OP_NODE_L2_ENGINE_AUTH} \
  --metrics=0.0.0.0:${METRICS_PORT} \
  --max-outbound-peers=200 \
  --chain ${RETH_CHAIN} \
  --rollup.sequencer-http=${RETH_SEQUENCER_HTTP} \
  --rollup.disable-tx-pool-gossip \
  --discovery.port=${DISCOVERY_PORT} \
  --port=${P2P_PORT}
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 5.2 创建 op-node.service

创建文件 `/etc/systemd/system/op-node.service`：

```ini
[Unit]
Description=Base OP Node (Consensus Layer)
After=network.target base-reth-node.service
Requires=base-reth-node.service

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/your-user/base-node
EnvironmentFile=/home/your-user/.base-node-env
ExecStartPre=/bin/bash -c 'until [ "$(curl -s --max-time 10 --connect-timeout 5 -w '\''%{http_code}'\'' -o /dev/null http://127.0.0.1:8551)" -eq 401 ]; do echo "等待执行层就绪..."; sleep 5; done'
ExecStart=/home/your-user/base-node/op-node/op-node/bin/op-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 5.3 启用并启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable base-reth-node
sudo systemctl enable op-node
sudo systemctl start base-reth-node
sudo systemctl start op-node

# 查看状态
sudo systemctl status base-reth-node
sudo systemctl status op-node

# 查看日志
sudo journalctl -u base-reth-node -f
sudo journalctl -u op-node -f
```

## 验证节点运行

### 检查 RPC 端点

```bash
# 检查执行层 RPC
curl -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# 检查 op-node RPC（如果启用）
curl -X POST http://127.0.0.1:7545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

### 检查指标

```bash
# 执行层指标
curl http://127.0.0.1:6060/metrics

# op-node 指标（如果启用）
curl http://127.0.0.1:7300/metrics
```

## 故障排除

### 常见问题

1. **JWT secret 文件不存在**
   - 确保 `OP_NODE_L2_ENGINE_AUTH` 指向的文件存在
   - 确保两个进程使用相同的 JWT secret

2. **端口冲突**
   - 检查端口是否被占用：`netstat -tulpn | grep <port>`
   - 修改环境变量中的端口配置

3. **执行层未就绪**
   - 检查 base-reth-node 日志
   - 确保数据目录有足够的空间和权限

4. **同步缓慢**
   - 考虑使用快照加速同步
   - 检查网络连接和 L1 RPC 端点性能

## 注意事项

- 确保两个进程（base-reth-node 和 op-node）使用**相同的 JWT secret**
- base-reth-node 必须在 op-node 之前启动
- 确保数据目录有足够的磁盘空间
- 定期备份数据目录
- 监控系统资源使用情况（CPU、内存、磁盘 I/O）

## 参考

- [Base 节点文档](https://docs.base.org/chain/run-a-base-node)
- [OP Stack 文档](https://docs.optimism.io/)
- [Reth 文档](https://reth.rs/)

