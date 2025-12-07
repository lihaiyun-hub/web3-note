# Geth 命令与参数

- 各命令的实现分布在：链管理 `cmd/geth/chaincmd.go`，账户与钱包 `cmd/geth/accountcmd.go`，控制台 `cmd/geth/consolecmd.go`，数据库工具 `cmd/geth/dbcmd.go`，快照工具 `cmd/geth/snapshot.go`，Verkle 工具 `cmd/geth/verkle.go`，配置导出 `cmd/geth/config.go`，版本与许可证 `cmd/geth/misccmd.go`
- 全局参数分组在应用启动时统一加入：`nodeFlags`、`rpcFlags`、`consoleFlags`、`debug.Flags`、`metricsFlags`
- 所有 CLI 标志支持环境变量映射，前缀为 `GETH_`

## 目录

- 命令概览
- 运行节点（默认行为）
- 链数据管理命令
- 数据库工具命令
- 快照工具命令
- Verkle 工具命令
- 账户与钱包命令
- 控制台与脚本命令
- 配置与杂项命令
- 全局参数说明（按类别分组）

---

## 命令概览

- 运行节点：无子命令时启动完整节点并阻塞运行
- 链数据管理：`init`、`dumpgenesis`、`import`、`export`、`import-history`、`export-history`、`import-preimages`、`dump`、`prune-history`、`download-era`
- 数据库工具：`removedb`、`db` 子命令（`inspect`、`stats`、`compact`、`get`、`delete`、`put`、`dumptrie`、`freezer-index`、`import`、`export`、`metadata`、`check-state-content`、`inspect-history`）
- 快照工具：`snapshot` 子命令（`prune-state`、`verify-state`、`check-dangling-storage`、`inspect-account`、`traverse-state`、`traverse-rawstate`、`dump`、`export-preimages`）
- Verkle 工具：`verkle` 子命令（`verify`、`dump`）
- 账户与钱包：`account`（`list`、`new`、`update`、`import`），`wallet import`
- 控制台与脚本：`console`、`attach`、`js`
- 配置与杂项：`dumpconfig`、`version`、`version-check`、`license`、`show-deprecated-flags`

---

## 运行节点（默认行为）

- 无子命令时执行 `geth`：创建并启动全节点，直到关闭
- 可通过全局参数控制网络、同步、RPC、缓存、挖矿、日志等（见下文“全局参数说明”）

示例：

- 主网全节点并开放 HTTP-RPC：
  - `geth --mainnet --http --http.api eth,net,web3`
- 自定义缓存与快照：
  - `geth --cache 4096 --snapshot`

---

## 链数据管理命令（`cmd/geth/chaincmd.go`）

- `geth init <genesisPath>`
  - 作用：初始化创世块与网络定义，破坏性操作，会重置当前网络。
  - 主要参数：
    - `--override.osaka`/`--override.bpo1`/`--override.bpo2`/`--override.verkle`：覆盖特定分叉时间戳（测试/开发用途）。
    - 数据库分组参数：`utils.DatabaseFlags`（见“数据库分组”）。
  - 用法：`geth init ./genesis.json --datadir ./mychain`

- `geth dumpgenesis`
  - 作用：打印当前网络的创世配置（若设置预设网络则打印对应预设，否则从 `datadir` 读取）。
  - 参数：`--datadir`，以及网络分组参数。

- `geth import <filename...>`
  - 作用：从 RLP 编码文件导入区块；支持多个文件。单文件导入失败即中止，多个文件导入会跳过失败条目。
  - 主要参数：缓存、快照、GC、度量、VM 跟踪、历史索引等（已在 Flags 列出）。以及 `utils.DatabaseFlags`、`debug.Flags`。

- `geth export <filename> [first last]`
  - 作用：导出链到 RLP 文件；支持指定区间；`.gz` 后缀启用压缩。
  - 参数：`--cache`、数据库分组。

- `geth import-history <dir>`
  - 作用：从 Era1 历史归档导入区块与收据；常用于合并前历史数据管理。
  - 参数：`history.transactions`、`txlookuplimit`（兼容）、数据库与网络分组。

- `geth export-history <dir> <first> <last>`
  - 作用：将区块与收据导出为 Era1 归档，通常以每 8192 区块打包。
  - 参数：数据库分组。

- `geth import-preimages <datafile>`
  - 作用：导入 Trie 键的哈希前像。
  - 参数：`--cache`、数据库分组。

- `geth dump [hash|number]`
  - 作用：按指定块或最新块转储状态；可排除代码/存储，控制起始键与最大元素。
  - 参数：`--nocode`、`--nostorage`、`--start`、`--limit`，数据库分组。

- `geth prune-history`
  - 作用：在合并块前裁剪历史区块体与收据，保留头；降低存储占用。
  - 参数：数据库分组。

- `geth download-era`
  - 作用：从 HTTP 源下载 Era1 历史（合并前）数据。
  - 参数：
    - `--block`：下载指定块或区间（`<start>-<end>`）。
    - `--epoch`：下载指定 epoch 或区间。
    - `--all`：下载所有可用 Era1 文件。
    - `--server`：Era1 服务器 URL。
    - 另含数据库与网络分组参数。

---

## 数据库工具命令（`cmd/geth/dbcmd.go`）

- `geth removedb`
  - 作用：删除区块链与状态数据库（带交互确认）。
  - 参数：
    - `--remove.state`：选择状态数据目录。
    - `--remove.chain`：选择古代链目录（冻结表）。
    - 数据库分组参数。

- `geth db inspect`（`cmd/geth/dbcmd.go:87`）
  - 作用：迭代数据库，按类型统计占用；可限定前缀与起始键。
  - 参数：网络与数据库分组。

- `geth db check-state-content [start]`
  - 作用：校验 Trie 节点键是否与值的 Keccak256 匹配，发现数据损坏。
  - 参数：网络与数据库分组。

- `geth db stats`
  - 作用：打印 LevelDB 统计信息。

- `geth db compact`
  - 作用：对 LevelDB 进行压缩；可能非常耗时、风险较高（中断可能导致损坏）。
  - 参数：`--cache`、`--cache.database`，网络与数据库分组。

- `geth db get <hex-key>` / `delete <hex-key>` / `put <hex-key> <hex-value>`
  - 作用：低层键值读写删除；操作不当可能破坏数据库。
  - 参数：网络与数据库分组。

- `geth db dumptrie <stateRoot> <accountHash> <storageRoot> [start] [max]`
  - 作用：导出指定存储 Trie 的键值对。

- `geth db freezer-index <freezer-type> <table-type> <start> <end>`
  - 作用：导出冻结表索引信息。

- `geth db import <dumpfile> [start]` / `export <type> <dumpfile>`
  - 作用：从/到 RLP 转储导入或导出链数据；`export` 支持 `.gz` 压缩。

- `geth db metadata`
  - 作用：显示链状态的元信息。

- `geth db inspect-history <address> [storage-slot]`
  - 作用：查询账户或存储槽在指定区块区间的状态历史；支持原始值展示。
  - 参数：`--start`、`--end`、`--raw`，网络与数据库分组。

---

## 快照工具命令（`cmd/geth/snapshot.go`）

- `geth snapshot prune-state <root>`
  - 作用：基于快照裁剪历史状态；删除不属于指定版本的 Trie 节点与合约代码；默认目标为 HEAD-127 状态。
  - 限制：仅支持 `state.scheme=hash`。
  - 参数：`--bloomfilter.size`、网络与数据库分组。

- `geth snapshot verify-state <root>`
  - 作用：基于快照遍历账户与存储，重算状态根做校验（快照→Trie 转换）。

- `geth snapshot check-dangling-storage <root>`
  - 作用：验证快照存储数据均有对应账户，避免“悬挂”。

- `geth snapshot inspect-account <address|hash>`
  - 作用：检查指定账户在所有快照层的情况并打印信息。

- `geth snapshot traverse-state <root>`
  - 作用：遍历状态并在缺失 Trie 节点或代码时中止；用于状态完整性校验。

- `geth snapshot traverse-rawstate <root>`
  - 作用：更细粒度的状态遍历与校验；与 `traverse-state` 类似但检查粒度更小。

- `geth snapshot dump [hash|number]`
  - 作用：语义等同 `geth dump`，但使用快照后端，性能更好。
  - 参数：`--nocode`、`--nostorage`、`--start`、`--limit`，网络与数据库分组。

- `geth snapshot export-preimages <dumpfile> [root]`
  - 作用：按快照枚举顺序导出前像，便于覆盖树迁移。

---

## Verkle 工具命令（`cmd/geth/verkle.go`）

- `geth verkle verify <root>`
  - 作用：根据给定根承诺重建并验证从 MPT 到 Verkle 的转换。

- `geth verkle dump <root> <key...>`
  - 作用：导出 Verkle 树为 DOT 文件，指定的键将被展开。

---

## 账户与钱包命令（`cmd/geth/accountcmd.go`）

- `geth account list`
  - 作用：打印所有存在账户的摘要。
  - 参数：`--datadir`、`--keystore`。

- `geth account new`
  - 作用：创建新账户，密钥以加密格式保存；交互式输入密码或通过 `--password` 非交互。
  - 参数：`--datadir`、`--keystore`、`--password`、`--lightkdf`。

- `geth account update <address>`
  - 作用：将账户迁移到最新加密格式或交互式修改密码；支持三次旧密码尝试。
  - 参数：`--datadir`、`--keystore`、`--lightkdf`。

- `geth account import <keyFile>`
  - 作用：从十六进制私钥文件导入为新账户；加密保存并提示密码。
  - 参数：`--datadir`、`--keystore`、`--password`、`--lightkdf`。

- `geth wallet import <presale.wallet>`
  - 作用：导入以太预售钱包，交互或 `--password` 非交互。
  - 参数：`--datadir`、`--keystore`、`--password`、`--lightkdf`。

---

## 控制台与脚本命令（`cmd/geth/consolecmd.go`）

- `geth console`
  - 作用：本地启动节点并附着 JS 控制台；支持 `--exec` 执行一段 JS 后退出；支持预加载脚本。
  - 参数：节点分组、RPC 分组、控制台分组。

- `geth attach [endpoint]`
  - 作用：连接到远程节点并打开 JS 控制台；不指定地址时默认 IPC 路径。
  - 参数：`--header/-H` 自定义 Header，控制台分组。

- `geth js <file...>`
  - 作用：已废弃；改用 `geth --exec "<JS语句>" console`。

---

## 配置与杂项命令

- `geth dumpconfig [dumpfile]`
  - 作用：导出当前配置为 TOML（默认输出至 stdout）。
  - 参数：节点与 RPC 分组；`--config` 指定 TOML 配置文件。
- `geth version`（`cmd/geth/misccmd.go:40-48`）
  - 作用：打印版本、Git 提交、架构、Go 版本、系统等；适合机器读取。
- `geth version-check [version]`
  - 作用：在线检查当前版本是否存在已知安全漏洞；数据源 URL 与版本可通过标志覆盖。
  - 参数：`--check.url`、`--check.version`。

---

## 全局参数说明（按类别分组）

> 以下标志定义集中在 `cmd/utils/flags.go` 与 `internal/debug/flags.go`，各命令通过其 `Flags` 字段组合使用。

### 数据库分组（DatabaseFlags）
- `--datadir`：数据与密钥存储目录。
- `--datadir.ancient`：古代数据根目录（冻结表），默认在 `chaindata` 内。
- `--datadir.era`：Era1 历史根目录（默认在 `ancient/chain` 内）。
- `--remotedb`：远程数据库 URL。
- `--db.engine`：数据库引擎（`pebble` 或 `leveldb`）。
- `--state.scheme`：状态存储方案（`hash` 或 `path`）。
- `--header/-H`：附加自定义 HTTP Header（与远程数据库或 `attach` 使用时）。

### 网络分组（NetworkFlags，`cmd/utils/flags.go:1024`）
- `--mainnet` / `--sepolia` / `--holesky` / `--hoodi`：选择预设网络；与 `--networkid` 与 `--override.genesis` 互斥。

### RPC/API 分组
- IPC：`--ipcdisable`、`--ipcpath`。
- HTTP：`--http`、`--http.addr`、`--http.port`、`--http.corsdomain`、`--http.vhosts`、`--http.api`、`--http.rpcprefix`。
- WS：`--ws`、`--ws.addr`、`--ws.port`、`--ws.api`、`--ws.origins`、`--ws.rpcprefix`。
- GraphQL：`--graphql`、`--graphql.corsdomain`、`--graphql.vhosts`（需启用 HTTP）。
- 认证 HTTP：`--authrpc.addr`、`--authrpc.port`、`--authrpc.vhosts`、`--authrpc.jwtsecret`。
- 保护与限额：`--rpc.gascap`、`--rpc.evmtimeout`、`--rpc.txfeecap`（全局交易费用上限）`--rpc.logquerylimit`、`--rpc.batch-request-limit`、`--rpc.batch-response-max-size`、`--rpc.allow-unprotected-txs`、`--rpc.txsync.defaulttimeout`、`--rpc.txsync.maxtimeout`。

### 控制台分组
- `--jspath`：`loadScript` 的 JS 根路径。
- `--exec`：执行一段 JS 语句后退出。
- `--preload`：在控制台预加载的 JS 文件清单。
- `--header/-H`：附加自定义 Header（远程控制台时）。

### 度量与监控分组
- `--metrics`：启用度量。
- `--metrics.addr`/`--metrics.port`：度量 HTTP 服务。
- InfluxDB(v1/v2) 上报：`--metrics.influxdb.*`、`--metrics.influxdbv2.*`（endpoint、token、bucket、org 等）。
- `--state.size-tracking`：启用状态大小跟踪（RPC `debug_stateSize`）。
- `--ethstats`：上报到 ethstats 服务。

### 日志与调试分组
- `--verbosity`：日志等级（0-5）。
- `--log.vmodule`：按模块设置日志等级（如 `eth/*=5,p2p=4`）。
- `--log.format`：日志格式（`json|logfmt|terminal`）。
- `--log.file`、`--log.rotate`、`--log.maxsize`、`--log.maxbackups`、`--log.maxage`、`--log.compress`：文件日志与滚动。
- pprof：`--pprof`、`--pprof.addr`、`--pprof.port`、`--pprof.memprofilerate`、`--pprof.blockprofilerate`、`--pprof.cpuprofile`。
- Go 执行跟踪：`--go-execution-trace`。

### 性能与缓存分组
- `--cache`：总体缓存（MB）。
- `--cache.database`：数据库 IO 缓存比例。
- `--cache.trie` / `--cache.gc` / `--cache.snapshot`：Trie/GC/快照缓存比例。
- `--cache.noprefetch`：禁用导入时状态预取。
- `--cache.preimages`：记录 Trie 键的哈希前像。
- `--cache.blocklogs`：日志过滤的块缓存大小。
- `--fdlimit`：提升进程文件描述符上限。
- `--crypto.kzg`：KZG 库选择（`gokzg|ckzg`）。

### 同步与历史分组
- `--syncmode`：同步模式（`snap|full`）。
- `--gcmode`：垃圾回收模式（`full|archive`）。
- 历史索引：`--history.transactions`、`--history.chain`（`all|postmerge`）、`--history.logs`、`--history.logs.disable`、`--history.logs.export`。
- `--exitwhensynced`：同步完成后自动退出。

### 网络与 P2P 分组
- `--port`：P2P 监听端口（默认 `30303`）。
- `--maxpeers` / `--maxpendpeers`：最大同行与最大挂起连接数。
- `--bootnodes`：引导节点列表（enode URLs）。
- `--nodekey`/`--nodekeyhex`：P2P 节点密钥（文件/十六进制）。
- `--nat`：NAT 端口映射机制（`any|none|upnp|pmp|extip:<IP>|stun:<IP:PORT>`）。
- `--nodiscover`：禁用自动发现（手动加节点）。
- `--discovery.v4`/`--discovery.v5`：启用 V4/V5 发现。
- `--netrestrict`：限制通信到给定 CIDR。
- `--discovery.dns`：设置 DNS 发现入口（空字符串禁用）。
- `--discovery.port`：自定义 UDP 发现端口。
- `--identity`：自定义节点名。

### 交易池与 Blob 池分组
- 交易池：`--txpool.locals`、`--txpool.nolocals`、`--txpool.journal`、`--txpool.rejournal`、`--txpool.pricelimit`、`--txpool.pricebump`、`--txpool.accountslots`、`--txpool.globalslots`、`--txpool.accountqueue`、`--txpool.globalqueue`、`--txpool.lifetime`。
- Blob 池：`--blobpool.datadir`、`--blobpool.datacap`、`--blobpool.pricebump`。

### 挖矿分组
- `--miner.gaslimit`：目标区块 gas 上限。
- `--miner.gasprice`：最小交易 gas 价格。
- `--miner.extradata`：区块额外数据（默认客户端版本）。
- `--miner.recommit`：重建待挖区块的间隔。
- `--miner.pending.feeRecipient`：待挖区块生产者地址（用于测试）。
- `--miner.maxblobs`：每区块最大 blobs 数（默认协议上限）。

### EVM/执行分组
- `--vmdebug`：记录 VM/合约调试信息。
- `--vmtrace`：选择内部操作跟踪器（开销大）。
- `--vmtrace.jsonconfig`：Tracer 的 JSON 配置。
- `--vmwitnessstats`：收集见证 Trie 访问统计（自动启用见证生成）。
- `--stateless-self-validation`：生成执行见证并进行自我校验（测试用途）。

### Beacon 轻客户端分组
- `--beacon.api`：CL 节点 API 地址（可多次）。
- `--beacon.api.header`：附加 HTTP Header（多次）。
- `--beacon.threshold`：同步委员会参与阈值。
- `--beacon.nofilter`：禁用未来插槽签名过滤。
- `--beacon.config`：Beacon 链配置 YAML 文件。
- `--beacon.genesis.gvroot`：Genesis validators root。
- `--beacon.genesis.time`：创世时间。
- `--beacon.checkpoint` / `--beacon.checkpoint.file`：弱主观性检查点（哈希/文件）。

### 账户分组
- `--password`：非交互式密码文件路径。
- `--signer`：外部签名器（URL 或 IPC）。
- `--keystore`：密钥库目录（默认在 datadir 内）。
- `--usb`：启用 USB 硬件钱包监控管理。
- `--pcscdpath`：智能卡守护进程（pcscd）socket 路径。
- `--lightkdf`：降低 KDF 强度以减少资源占用（测试环境）。

### 杂项分组
- `--override.genesis`：从文件加载创世与配置（与预设网络互斥）。
- `--eth.requiredblocks`：强制对等校验的区块号→哈希映射。
- `--synctarget`：指定需要全同步到的区块哈希（开发测试用途）。

---

## 用法示例

- 启动主网并开放 HTTP-RPC：
  - `geth --mainnet --http --http.api eth,net,web3`
- 自定义链初始化：
  - `geth init ./genesis.json --datadir ./mychain`
- 导入区块转储：
  - `geth import blocks.rlp --datadir ./mychain`
- 本地控制台并预加载脚本：
  - `geth console --preload ./scripts/setup.js`
- 远程 IPC 附着：
  - `geth attach ~/.ethereum/geth.ipc`
- 导出当前配置为 TOML：
  - `geth dumpconfig geth.toml`



