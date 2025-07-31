
# Monallo Bridge 产品部署总览

## 1. 项目概述与架构

**Monallo Bridge** 是一套完整的跨链资产桥接解决方案，实现不同区块链网络之间安全、高效的资产转移。用户可在一条链（源链）锁定原生或 ERC20 代币，并在另一条链（目标链）铸造封装代币（wrapped token），反之亦然。

### 整体技术架构

Monallo Bridge 由以下四个核心组件构成：

#### 智能合约 (Smart Contracts)
- **Source.sol**：部署在源链上，负责锁定资产、处理手续费、验证解锁。
- **Target.sol**：部署在目标链上，负责铸造和销毁封装代币。

#### 中间件 (Middleware / Relayer)
- 系统的核心引擎。这是一个链下 Node.js 服务，扮演着**中继器 (Relayer)** 的角色。它持续监听源链和目标链上的合约事件 (`AssetLocked, TokensBurned`)，验证这些事件，并使用其签名者私钥在另一条链上发起相应的交易 (`mint, unlock`)，从而完成跨链通信。


#### 后端 (Backend)
- 一个基于 Node.js 的 API 服务，使用 MongoDB 数据库。它负责**记录和查询**所有跨链交易的历史数据，管理用户信息，并为前端提供数据支持。它不直接参与链上交易，而是作为数据和业务逻辑层。


#### 前端 (Frontend)
- 用户与跨链桥交互的 Web 界面。用户通过前端连接自己的钱包（如 MetaMask），发起资产锁定或销毁请求。前端与后端 API 通信以获取交易历史，并与链上智能合约交互以执行用户的交易。

### 工作流程示例（源链 → 目标链）
1. 用户在前端发起一笔从`以太坊`到`IMUA`的资产转移；
2. 前端调用以太坊 Source 合约的 `lock` 方法；
3. 合约锁定资产并触发事件；
4. 中间件监听事件、验证、签名并在 IMUA 链上调用 `mint`；
5. Target 合约铸造封装代币；
6. 后端记录交易，供前端查询。

---

## 2. 组件部署指南

本指南将按照逻辑依赖顺序，依次介绍如何部署 Monallo Bridge 的各个组件。推荐的部署顺序为：**智能合约 → 后端 → 中间件 → 前端**。

---

## a. 智能合约部署 (Smart Contracts)

#### 前提条件
- Remix IDE
- 已连接 MetaMask 钱包
- 准备中继器签名者地址（Relayer Signer Address）

#### 1. 准备合约代码
**Source.sol**: https://github.com/MonalloAI/Monallo-Bridge-Contract/blob/main/contracts/double-bridge/v0.2/Source.sol  
**Target.sol**: https://github.com/MonalloAI/Monallo-Bridge-Contract/blob/main/contracts/double-bridge/v0.2/Target.sol

#### 2. 编译合约
- 使用 Solidity 编译器 v0.8.19 
- 编译 Source.sol 和 Target.sol，确保无错误

#### 3. 部署 Source.sol（源链）
- 切换网络至源链（如 Sepolia）
- 选择合约并点击 Deploy，无需构造参数
- 在 MetaMask 中确认交易，记录合约地址

#### 4. 配置 Source.sol
部署完成后，需要对 Source 合约进行关键配置。操作者默认为合约管理员。
- **设置 Relayer 签名者 (setRelayerSigner)**
  - 调用 `setRelayerSigner` 并输入 Relayer 地址，此地址必须与后续部署 Target 合约及中间件配置中的地址一致。
- **设置桥接费用 (setFeeConfig)**
  - 调用 `setFeeConfig`
      *   `_isPercentage` (bool): 如果费用是百分比，则为 `true`；如果是固定值，则为 `false`。
    *   `_value` (uint256):
        *   如果 `_isPercentage` 为 `true`，`_value` 表示基点（basis points），即万分之一。例如，`100` 表示 `1%` (`100/10000`)。最大值为 `10000` (`100%`)。
        *   如果 `_isPercentage` 为 `false`，`_value` 表示以 wei 为单位的固定费用。

#### 5. 部署 Target.sol（目标链）
Target.sol 合约部署在资产接收的链上（例如，IMUA 测试网）。
- 切换网络至 IMUA 测试网
- 在 Deploy 按钮旁边的输入框中，依次填入以下构造函数参数：
  - `name_ (string)`: 封装代币的名称 (e.g., "Monallo ETH")。
  - `symbol_ (string)`: 封装代币的符号 (e.g., "maoETH")。
  - `sourceChainId_ (uint256)`: 源链的 Chain ID (e.g., Sepolia 为 11155111)。
  - `relayerSigner_ (address)`: 与 Source 合约中设置的中继器签名者地址保持一致。
- 示例输入格式: `"Monallo ETH", "maoETH", 11155111, "0xYourRelayerSignerAddress"`
- 点击` Deploy `按钮并在 MetaMask 中确认交易，部署并记录合约地址

#### 6. 配置 Target.sol 权限
部署者默认拥有 DEFAULT_ADMIN_ROLE 和 MINTER_ROLE。您也可以根据需要授予其他地址相应角色。最重要的是，确保中继器拥有铸币权限。
-  管理角色 (grantRole):
   - `MINTER_ROLE`: 只有拥有此角色的地址（即中继器）才能调用 `mint` 函数。如果部署时使用的地址不是中继器地址，需要将此角色授予中继器签名者地址。

1.在合约实例中找到` MINTER_ROLE `变量，点击获取其角色哈希值。
2.调用`grantRole `函数，`role` 填入角色哈希，`account` 填入中继器签名者地址。

`重要`: 请务必妥善保管所有已部署的合约地址 (`Source` 和 `Target`) 以及中继器签名者地址和私钥。这些信息将在后续组件的配置中使用。


---

## b. 后端部署 (Backend)
后端服务负责提供 API 接口，用于存储和查询跨链交易记录。
#### 环境要求
- Node.js >= 16.0.0
- MongoDB >= 4.4.0
- npm >= 8.0.0
#### 1. 克隆项目与安装依赖
```bash
#克隆仓库
git clone https://github.com/MonalloAI/Monallo-Bridge-Backend.git
cd Monallo-Bridge-Backend
#安装依赖
npm install
```

#### 2. 环境变量 
在项目根目录创建 `.env` 文件，并填入以下配置：
```env
# 服务器配置
PORT=5000
NODE_ENV=production

# 数据库配置
MONGO_URI=mongodb://localhost:27017/monallo_bridge
# 或者使用 MongoDB Atlas
# MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/monallo_bridge

# 区块链配置
SOURCE_CHAIN_ID=1
TARGET_CHAIN_ID=137
SOURCE_RPC_URL=https://mainnet.infura.io/v3/YOUR_INFURA_KEY
TARGET_RPC_URL=YOUR_TARGET_RPC_URL

# 安全配置
JWT_SECRET=your_jwt_secret_key
CORS_ORIGIN=http://localhost:3000

# 其他配置
LOG_LEVEL=info
```

#### 3. 启动服务
确保 MongoDB 服务正在运行。
```bash
# Ubuntu/Debian
sudo systemctl start mongod

# macOS (使用 Homebrew)
brew services start mongodb-community

# Windows
net start MongoDB
```
- 开发模式：
```bash
npm run dev
```
- 生产模式：
```bash
# 构建项目
npm run build

# 启动服务
npm start
```
### 使用 PM2 管理进程

```bash
# 安装 PM2
npm install -g pm2

# 启动服务
pm2 start dist/server.js --name "monallo-bridge-backend"

# 查看状态
pm2 status

# 查看日志
pm2 logs monallo-bridge-backend

# 重启服务
pm2 restart monallo-bridge-backend

# 停止服务
pm2 stop monallo-bridge-backend
```
## API 接口说明

### 基础路径
- 开发环境: `http://localhost:5000/api`
- 生产环境: `https://your-domain.com/api`

### 主要接口

#### 跨链记录管理
- `GET /allCrossRecords` - 获取所有跨链记录
- `GET /getCrossRecordByTxHash?sourceFromTxHash=xxx` - 根据交易哈希获取记录
- `GET /getCrossRecordByFromAddress?from=xxx` - 根据地址获取记录
- `GET /getCrossLockByTxHash?sourceFromTxHash=xxx` - 获取锁定记录
- `GET /getCrossLockByFrom?from=xxx` - 根据地址获取锁定记录

#### 用户管理
- `POST /users/register` - 用户注册
- `POST /users/login` - 用户登录
- `GET /users/profile` - 获取用户信息

#### 授权验证
- `POST /auth/verify` - 验证用户权限
- `GET /auth/status` - 获取授权状态

#### 日志管理
- `GET /logs` - 获取系统日志
- `POST /logs` - 创建日志记录

## 数据库结构

### 主要集合

1. **CrossBridgeRecord** - 跨链记录
   - `sourceChainId, sourceChain, sourceRpc`
   - `sourceFromAddress, sourceFromTokenName`
   - `sourceFromAmount, sourceFromTxHash`
   - `targetChainId, targetChain, targetRpc`
   - `targetToAddress, targetToTokenName`
   - `crossBridgeStatus, createdAt, updatedAt`

2. **User** - 用户信息
   - `username, email, password`
   - `role, status, createdAt`

3. **Authorization** - 授权记录
   - `userId, token, expiresAt`

4. **Log** - 系统日志
   - `level, message, timestamp, userId`

## 监控和维护

### 日志管理

```bash
# 查看应用日志
tail -f logs/app.log

# 查看错误日志
tail -f logs/error.log

# 使用 PM2 查看日志
pm2 logs monallo-bridge-backend --lines 100
```

### 性能监控

```bash
# 查看进程状态
pm2 monit

# 查看系统资源使用
htop
```

### 数据库备份

```bash
# 备份数据库
mongodump --db monallo_bridge --out /backup/$(date +%Y%m%d)

# 恢复数据库
mongorestore --db monallo_bridge /backup/20231201/monallo_bridge
```

## 故障排除

### 常见问题

1. **MongoDB 连接失败**
   - 检查 MongoDB 服务是否启动
   - 验证 MONGO_URI 配置是否正确
   - 检查网络连接和防火墙设置

2. **端口被占用**
   ```bash
   # 查看端口占用
   lsof -i :5000
   
   # 杀死占用进程
   kill -9 <PID>
   ```

3. **内存不足**
   - 增加服务器内存
   - 优化数据库查询
   - 使用 PM2 进行进程管理

4. **TypeScript 编译错误**
   ```bash
   # 清理并重新安装依赖
   rm -rf node_modules package-lock.json
   npm install
   npm run build
   ```

### 健康检查

创建健康检查接口：

```typescript
// 在 app.ts 中添加
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage()
  });
});
```

## 安全建议

1. **环境变量管理**
   - 不要在代码中硬编码敏感信息
   - 使用环境变量存储配置
   - 定期轮换密钥和密码

2. **数据库安全**
   - 启用 MongoDB 认证
   - 限制数据库访问权限
   - 定期备份数据

3. **网络安全**
   - 配置防火墙规则
   - 使用 HTTPS
   - 实施 CORS 策略

4. **应用安全**
   - 输入验证和清理
   - 防止 SQL 注入
   - 实施速率限制

## 更新部署

### 代码更新流程

```bash
# 1. 拉取最新代码
git pull origin main

# 2. 安装新依赖
npm install

# 3. 构建项目
npm run build

# 4. 重启服务
pm2 restart monallo-bridge-backend

# 5. 验证服务状态
pm2 status
curl http://localhost:5000/health
```

### 回滚策略

```bash
# 回滚到上一个版本
git reset --hard HEAD~1
npm run build
pm2 restart monallo-bridge-backend
```

---

## c. 中间件部署 (Middleware)
中间件是连接源链和目标链的桥梁，负责自动处理跨链交易。`这是系统中最关键的组件`。
#### 环境要求
- Node.js >= 18.0.0
- MongoDB >= 4.4.0

#### 1. 克隆项目与安装依赖
```bash
# 克隆仓库
git clone https://github.com/MonalloAI/Monallo-Bridge-Middleware.git
cd Monallo-Bridge-Middleware

# 安装依赖
npm install
```

#### 2. 环境变量
创建 `.env` 文件并配置以下环境变量：
```env
# 私钥配置（用于签名交易）
PRIVATE_KEY=your_private_key_here

# 数据库配置
MONGO_URI=mongodb://localhost:27017/monallo_bridge

# 区块链 RPC 配置
IMUA_RPC_URL=wss://your_imua_rpc_url
ETH_RPC_URL=wss://eth-sepolia.g.alchemy.com/v2/
ETH_API_KEY=your_alchemy_api_key
PLATON_RPC_URL=https://openapi2.platon.network/rpc

# 服务端口
PORT=3001
```

#### 3. 配置合约地址
确保 `src/abi/deployed_addresses.json` 文件中的合约地址正确：

```json
{
  "TOKEN_CONTRACTS": {
    "Ethereum-Sepolia": {
      "ETH": "",
      "USDC": "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
      "EURC": "0x08210f9170f89ab7658f0b5e3ff39b0e03c594d4",
      "maoIMUA": "0x12306381b1b6ecb4132ff4ce324ed2be3728e865",
      "maoZETA": "0x13864cc6Ac76F4109254D6C2ED90807a2904563A",
      "maoUSDC": "0x7562c0d1ee790aed045839aee88d2e29fdf010d2",
      "maoLAT": "0x1afd2d6f77b125b2b18c471f7ba95b009a039ba8"
    },
    "Imua-Testnet": {
      "IMUA": "",
      "maoETH": "0x4a91a4a24b6883dbbddc6e6704a3c0e96396d2e9",
      "maoLAT": "0x924A9fb56b2b1B5554327823b201b7eEF691E524",
      "maoZETA": "0xFCE1AC30062EfDD9119F6527392D4B935397f714",
      "maoEURC": "0xDFEc8F8C99eC22AA21e392Aa00eFb3F517C44987",
      "maoUSDC": {
        "PlatON": "0x4ed64b15ab26b8fe3905b4101beccc1d5b3d49fd",
        "Ethereum-Sepolia": "0xe5a26a2c90b6e629861bb25f10177f06720e5335"
      }
    }
  },
  "LOCK_CONTRACTS": {
    "Ethereum-Sepolia": "0x4195868b54b70d4420E6203e85A4b92a6705FF28",
    "PlatON-Mainnet": "0x2fd92027B1afB80613B5720Df1015D41873F8d7C",
    "Imua-Testnet": "0xfcc4936B0b437469F5CE4C3cBD7eaa05CE5f581d",
    "ZetaChain-Testnet": "0x1870f6D7A02994EE08E7c9BC3aEad81f00de1A05"
  }
}
```

#### 4. 启动服务
开发模式：
```bash
npm run dev
```
生产模式：
```bash
# 构建项目
npm run build

# 启动服务
npm start
```
手动队列检查：
```bash
npm run check-queue
```
#### 5. 服务管理
##### 1.使用 PM2 管理服务
```bash
# 安装 PM2
npm install -g pm2

# 启动服务
pm2 start dist/server.js --name "monallo-bridge"

# 查看状态
pm2 status

# 查看日志
pm2 logs monallo-bridge

# 重启服务
pm2 restart monallo-bridge

# 停止服务
pm2 stop monallo-bridge
```

##### 2.使用 systemd 管理服务
创建服务文件 `/etc/systemd/system/monallo-bridge.service`：

```ini
[Unit]
Description=Monallo Bridge Middleware
After=network.target

[Service]
Type=simple
User=your_user
WorkingDirectory=/path/to/Monallo-Bridge-Middleware
ExecStart=/usr/bin/node dist/server.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

启动服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable monallo-bridge
sudo systemctl start monallo-bridge
sudo systemctl status monallo-bridge
```

#### 配置说明
##### 环境变量详解

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `PRIVATE_KEY` | 用于签名交易的私钥 | `0x1234...` |
| `MONGO_URI` | MongoDB 连接字符串 | `mongodb://localhost:27017/monallo_bridge` |
| `IMUA_RPC_URL` | Imua 网络 WebSocket RPC | `wss://rpc.imua.network` |
| `ETH_RPC_URL` | Ethereum Sepolia WebSocket RPC | `wss://eth-sepolia.g.alchemy.com/v2/` |
| `ETH_API_KEY` | Alchemy API 密钥 | `your_api_key` |
| `PLATON_RPC_URL` | PlatON 网络 HTTP RPC | `https://openapi2.platon.network/rpc` |
| `PORT` | 服务端口 | `3001` |

#### 网络配置

##### 支持的区块链网络
1. **Ethereum Sepolia** (Chain ID: 11155111)
   - 测试网络，用于开发和测试
   - 支持 ETH、USDC、EURC 等代币

2. **PlatON Mainnet** (Chain ID: 210425)
   - PlatON 主网
   - 支持 LAT、USDC 等代币

3. **Imua Testnet** (Chain ID: 233)
   - Imua 测试网络
   - 支持 IMUA 原生代币和各种锚定代币

4. **ZetaChain Testnet** (Chain ID: 7001)
   - ZetaChain 测试网络
   - 支持 ZETA 原生代币和各种锚定代币

#### 代币映射

##### 锚定代币映射
- `maoETH` ↔ `ETH` (Ethereum Sepolia)
- `maoLAT` ↔ `LAT` (PlatON Mainnet)
- `maoZETA` ↔ `ZETA` (ZetaChain Testnet)
- `maoUSDC` ↔ `USDC` (各网络)
- `maoEURC` ↔ `EURC` (各网络)

#### 代币精度处理
- **USDC**: 6位小数 → 18位小数（铸造时）
- **其他代币**: 18位小数

#### 6.监控和维护

##### 1.日志监控

- 查看实时日志
```bash
# 使用 PM2
pm2 logs monallo-bridge --lines 100

# 使用 systemd
sudo journalctl -u monallo-bridge -f
```
##### 日志级别 
- `✅` - 成功操作
- `❌` - 错误信息
- `⚠️` - 警告信息
- `🔄` - 重试/重连操作
- `🔔` - 事件监听
- `💰` - 金额相关
- `🔐` - 签名相关

##### 2. 数据库监控

- 连接状态检查
```bash
# 进入 MongoDB shell
mongo
use monallo_bridge
db.stats()
```

- 跨链记录查询
```javascript
// 查询所有跨链记录
db.crossbridgerecords.find()

// 查询失败的记录
db.crossbridgerecords.find({crossBridgeStatus: "failed"})

// 查询待处理的记录
db.crossbridgerecords.find({crossBridgeStatus: "pending"})
```

#### 7. 网络连接监控

##### WebSocket 连接状态
中间件会自动监控各网络的 WebSocket 连接状态：
- 每5分钟检查一次连接状态
- 自动重连断开的连接
- 重连后自动检查待处理队列

##### 队列检查
- 每30分钟自动检查待处理队列
- 手动检查：`npm run check-queue`

#### 8. 性能监控

- 内存使用
```bash
# 查看 Node.js 进程内存使用
ps aux | grep node

# 使用 PM2 监控
pm2 monit
```

- 网络延迟
监控各区块链网络的 RPC 响应时间，确保网络连接稳定。

## 故障排除

### 常见问题

#### 1. 数据库连接失败
**症状**: `MongoDB connection error`
**解决方案**:
```bash
# 检查 MongoDB 服务状态
sudo systemctl status mongod

# 重启 MongoDB
sudo systemctl restart mongod

# 检查连接字符串
echo $MONGO_URI
```

#### 2. 私钥错误
**症状**: `Invalid private key`
**解决方案**:
- 确保私钥格式正确（0x开头）
- 检查私钥是否有足够的余额支付 gas 费用

#### 3. RPC 连接失败
**症状**: `WebSocket connection failed`
**解决方案**:
- 检查网络连接
- 验证 RPC URL 是否正确
- 检查 API 密钥是否有效

#### 4. 合约地址错误
**症状**: `Contract not found`
**解决方案**:
- 检查 `deployed_addresses.json` 中的合约地址
- 确认合约已正确部署
- 验证 ABI 文件是否正确

#### 5. Gas 费用不足
**症状**: `INSUFFICIENT_FUNDS`
**解决方案**:
- 向钱包地址充值足够的原生代币
- 检查 gas 价格设置

### 调试模式

#### 启用详细日志
在 `.env` 文件中添加：
```env
DEBUG=true
NODE_ENV=development
```

#### 手动测试
```bash
# 测试队列检查
npm run check-queue

# 测试特定网络连接
npm run test-queue
```

### 安全考虑

#### 1. 私钥安全
- 使用环境变量存储私钥
- 不要将私钥提交到代码仓库
- 定期轮换私钥
- 使用硬件钱包或安全的密钥管理服务

#### 2. 网络安全
- 使用 HTTPS/WSS 连接
- 配置防火墙规则
- 定期更新依赖包
- 监控异常访问

#### 3. 数据库安全
- 启用 MongoDB 认证
- 限制数据库访问权限
- 定期备份数据
- 监控数据库访问日志

### 更新和维护

#### 1. 代码更新
```bash
# 拉取最新代码
git pull origin main

# 安装新依赖
npm install

# 重新构建
npm run build

# 重启服务
pm2 restart monallo-bridge
```

#### 2. 依赖更新
```bash
# 检查过时的依赖
npm outdated

# 更新依赖
npm update

# 更新特定包
npm install package-name@latest
```

#### 3. 数据库备份
```bash
# 创建备份
mongodump --db monallo_bridge --out /backup/$(date +%Y%m%d)

# 恢复备份
mongorestore --db monallo_bridge /backup/20231201/monallo_bridge/
```


---

## d. 前端部署 (Frontend)
前端是用户进行跨链操作的界面。
#### 环境要求
- 操作系统：Windows / macOS / Linux 均可  
- Node.js：建议版本 v16 及以上（请先安装 Node.js）  
- 包管理工具：npm（Node.js 自带）或 yarn  
- Git：用于克隆项目代码  

检查 Node.js 与 npm 版本：

```
node -v
npm -v
```
#### 1. 克隆项目与安装依赖
```bash
# 克隆仓库
git clone https://github.com/MonalloAI/Monallo-Bridge-Frontend.git
cd Monallo-Bridge-Frontend

# 安装依赖
npm install
```


#### 3. 启动与构建
- 本地开发/调试 ：
```bash
npm run dev
```
本地访问地址通常是 `http://localhost:3000`，具体启动地址可以看控制台输出。
- 生产构建：
```bash
npm run build
```



---

## ✅ 部署完成

完成上述所有步骤后，您的 Monallo Bridge 实例已成功可供使用。
