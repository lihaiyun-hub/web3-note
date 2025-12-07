# Go + abigen 合约部署完整指南

## 环境准备
- 安装 `solc`（推荐 `0.8.20` 或兼容版本）
  - 用途：编译 Solidity 源码，生成 ABI/BIN/metadata。
  - 安装与检查（Linux 示例，二选一）：
    ```bash
    # 选项 A：Snap（常见于 Ubuntu）
    sudo snap install solc --classic
    solc --version
    
    # 选项 B：Ubuntu PPA
    sudo add-apt-repository ppa:ethereum/ethereum -y
    sudo apt update
    sudo apt install -y solc
    solc --version
    ```
- 安装 `abigen`（与仓库一致的版本）
  - 用途：从 ABI/BIN 或 `combined-json` 生成 Go 绑定。
  - 安装命令：
    ```bash
    go install github.com/ethereum/go-ethereum/cmd/abigen@v1.16.7
    ```
- 安装 OpenZeppelin 依赖（供 `solc` 解析 import）
  - 用途：提供 `ERC20`, `Ownable`, `Pausable` 等实现。
  - 安装命令：
    ```bash
    npm install @openzeppelin/contracts
    node -v && npm -v
    ```
 - 检查 Go 版本与 abigen 可用性：
   ```bash
   go version
   which abigen 
   ```

## 编译与生成绑定（两种方式）
### 方式 A：组合输出（推荐）
1) 使用 `solc` 生成综合工件：
```bash
mkdir -p build
solc \
  --optimize \
  --combined-json abi,bin,metadata \
  --base-path . \
  --include-path node_modules \
  ethclient-demo/sol/mytoken.sol > build/combined.json
```
- `--optimize`：启用优化，通常降低 gas 或字节码体积。
- `--combined-json abi,bin,metadata`：一次性生成 ABI、BIN 与 metadata，便于 `abigen` 直接读取。
- `--base-path .`：设置当前目录为 import 基准路径。
- `--include-path node_modules`：允许解析 `@openzeppelin/contracts/...`。
- 重定向到 `build/combined.json`：集中保存所有合约的工件。

2) 使用 `abigen` 生成 Go 绑定：
```bash
abigen \
  --combined-json build/combined.json \
  --pkg token \
  --out ethclient-demo/token/mytoken_bindings.go
```
- `--combined-json`：指向上一步生成的综合工件。
- `--pkg token`：生成的 Go 包名，建议与目录统一，便于 `import "ethclient-demo/token"`。
- `--out`：绑定文件输出路径。

### 方式 B：分离输出（备用）
1) 生成独立 ABI 与 BIN 文件：
```bash
mkdir -p build
solc \
  --optimize \
  --abi --bin \
  --base-path . \
  --include-path node_modules \
  -o build \
  ethclient-demo/sol/mytoken.sol
```
- `-o build`：将所有生成文件写入 `build/` 目录。
- 输出示例：`build/MyToken.abi` 与 `build/MyToken.bin`。

2) 从 ABI/BIN 生成 Go 绑定：
```bash
abigen \
  --abi build/MyToken.abi \
  --bin build/MyToken.bin \
  --pkg token \
  --out ethclient-demo/token/mytoken_bindings.go
```
- `--abi/--bin`：分别指向目标合约的 ABI 与字节码。

## 部署代码核心流程（参数逐项说明）
部署示例见 `ethclient-demo/10_deploycontract.go:17-77`。以下为关键步骤与参数含义：

- 连接 RPC 与加载私钥
  - `ethclient.Dial(rpcURL)`：连接到以太坊节点 RPC（例如 Infura、Alchemy 或自建节点）。
  - `crypto.HexToECDSA(privateHex)`：从十六进制字符串加载私钥。生产环境请从环境变量或 KMS 读取，避免明文写入源码。
  - 执行命令（推荐使用环境变量）：
    ```bash
    export ETH_RPC_URL="https://sepolia.infura.io/v3/<YOUR_PROJECT_ID>"
    export PRIVATE_KEY_HEX="<YOUR_PRIVATE_KEY_HEX>"
    ```

- 网络与账户信息
  - `client.NetworkID(ctx)`：获取链 ID（EIP‑155），用于正确签名交易。
  - `from := crypto.PubkeyToAddress(privateKey.PublicKey)`：推导部署者地址。通常将其同时作为 `initialOwner` 与 `withdrawAddress`。

- 动态费用模型（伦敦后）
  - `nonce := client.PendingNonceAt(ctx, from)`：获取待用 nonce，避免重复或覆盖交易。
  - `tip := client.SuggestGasTipCap(ctx)`：建议的优先费（`GasTipCap`，给打包者的小费）。
  - `hdr := client.HeaderByNumber(ctx, nil)`：最新块头，取 `BaseFee`。
  - `feeCap := 2*BaseFee + tip`：总费用上限 `GasFeeCap` 的经验值，可根据网络拥堵调整。

- 构造交易选项与设置费用
  - `bind.NewKeyedTransactorWithChainID(privateKey, chainID)`：构造带链 ID 的 `TransactOpts`。
  - 重要字段：
    - `auth.Nonce = big.NewInt(int64(nonce))`
    - `auth.GasTipCap = tip`
    - `auth.GasFeeCap = feeCap`
    - `auth.GasLimit = 1200000`：部署通常需要较高上限，建议结合 `EstimateGas` 校准。

- 部署参数与调用
  - Solidity 构造函数：`constructor(address initialOwner, address _withdrawAddress)`（ethclient-demo/sol/mytoken.sol:1-20）
  - Go 绑定调用：`addr, tx, _, err := token.DeployMyToken(auth, client, initialOwner, withdrawAddress)`（ethclient-demo/token/mytoken_bindings.go:2677）
  - 返回值：合约地址、部署交易、绑定实例；记录 `tx.Hash()` 与 `addr.Hex()` 便于链上跟踪。
  - 执行部署（运行程序）：
    ```bash
    # 在 ethclient-demo/main.go 中调用 DeployContract() 后
    go run .
    ```

- 余额与费用校验（避免失败）
  - `bal := client.BalanceAt(ctx, from, nil)`：查询当前余额。
  - `estFee := GasLimit * GasFeeCap`：粗略估算最大可能消耗，余额不足则先充值或降低参数。

## 关键命令与参数速查
- `solc`：
  - `--optimize`：启用编译优化。
  - `--combined-json abi,bin,metadata`：生成综合工件（推荐）。
  - `--abi --bin`：分别生成 ABI 与 BIN 文件（备用）。
  - `--base-path .`：设置 import 基准路径。
  - `--include-path node_modules`：允许解析 `@openzeppelin/contracts/...`。
  - `-o build`：输出目录。

- `abigen`：
  - `--combined-json <file>`：从综合工件生成绑定。
  - `--abi <file> --bin <file>`：从分离工件生成绑定。
  - `--pkg <name>`：Go 包名。
  - `--out <path>`：输出文件路径。

- 部署函数（Go 绑定）：
  - `DeployMyToken(auth, backend, initialOwner, _withdrawAddress)`（ethclient-demo/token/mytoken_bindings.go:2677）
    - `auth`：交易签名与费用参数（`TransactOpts`）。
    - `backend`：`bind.ContractBackend`，通常是 `*ethclient.Client`。
    - `initialOwner`：合约所有者地址。
    - `_withdrawAddress`：提现接收地址，不能为零地址。

- 动态费用（伦敦）相关：
  - `GasTipCap`：优先费，影响打包速度。
  - `GasFeeCap`：总费用上限，应大于 `BaseFee + GasTipCap`。
  - `GasLimit`：部署交易的 gas 上限。

## 部署后验证与交互
- 等待上链并获取回执：
  ```go
  receipt, err := bind.WaitMined(ctx, client, tx)
  ```
- 加载合约并进行读方法调用示例：
  - 参考：`ethclient-demo/11_loadcontract.go:14-41`
  - 从地址加载：`token.NewMyToken(common.HexToAddress(addr), client)`
  - 读余额：`myToken.BalanceOf(&bind.CallOpts{Context: ctx}, holder)`
  - 执行读取（运行程序）：
    ```bash
    export TOKEN_ADDRESS="<DEPLOYED_CONTRACT_ADDRESS>"
    export ADDRESS="<HOLDER_ADDRESS>"
    go run .
    ```

## 常见问题与排查
- `solc` 报错：找不到 `@openzeppelin/contracts/...`
  - 确认已执行 `npm install @openzeppelin/contracts`。
  - OpenZeppelin v5 的 `Pausable` 路径为 `utils/Pausable.sol`。
- 部署失败或余额不足
  - 提高 `GasTipCap` 或 `GasFeeCap`；检查 `estFee` 是否超过余额。
  - 确认 `chainID` 与目标网络匹配，RPC 是否可用。
- 交易长期 `pending`
  - 提高 `GasTipCap`；检查当前 `BaseFee` 与设置是否偏低；可考虑替换交易策略（相同 nonce 更高费用）。
- 构造参数错误
  - `_withdrawAddress` 不可为零地址（合约内有 `require` 检查）。

## 安全与生产建议
- 私钥管理
  - 不要将明文私钥写入源码或提交仓库。示例代码仅为演示用途，应改为读取环境变量或使用 KMS/HSM。
  - 环境变量清理：
    ```bash
    unset PRIVATE_KEY_HEX
    ```
- 费用策略
  - 生产环境结合历史平均与当前拥堵动态设置 `GasFeeCap`；避免过低导致挂起，或过高造成成本浪费。
- 错误处理与重试
  - 使用 `context.Context` 控制超时；记录并上报失败原因；对可重试错误（网络抖动、RPC 超时）进行指数退避重试。

