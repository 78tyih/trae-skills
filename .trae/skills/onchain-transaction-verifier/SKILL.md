---
name: "onchain-transaction-verifier"
description: "链上交易真实性交叉核验 Skill。用于核验“已转账但未到账”“共享屏幕显示成功”“钱包 UI/截图显示成功”“TxID/区块高度可疑”等场景。输入 TxHash、付款地址 From、收款地址 To、Block Height、Timestamp 中任意已知字段，并结合 Amount/Token 等辅助信息，从公开区块链账本反推出其余字段并检查是否互相一致。TRON/TRC20 为首个重点支持网络，方法可扩展至 EVM/Solana。"
---

# On-chain Transaction Verifier — 反诈骗查链 Skill

## 目标

这个 Skill 只回答一个可独立验证的问题：

> **对方声称的链上交易，是否真实存在于公开账本，并且时间、区块、哈希、付款地址、收款地址等信息能够互相对应？**

不要先判断“某个人是不是骗子”。先把对方的说法拆成链上字段，再让公开账本回答。

---

## 核心原则

区块链是一套公开账本。钱包截图、共享屏幕、绿色 Success、二维码、甚至格式正确的 64 位哈希都只是“对方展示的信息”；**真正的事实源是主网节点、公开区块浏览器和原始链上数据。**

核心五项：

1. **TxHash / TxID** — 交易哈希
2. **From** — 付款地址
3. **To** — 收款地址
4. **Block Height** — 区块高度
5. **Timestamp** — 交易时间

强辅助项：

- **Amount** — 金额
- **Token / Contract** — 币种及代币合约
- Network / Chain — 链和网络
- Status / Receipt — 执行状态

> 通常拿到核心五项中的一个强锚点（尤其 TxHash），或两项可组合线索，就可以开始反查并补全其它字段。  
> “两项足够”不是数学保证：地址 + 金额等组合可能对应多笔交易，此时必须增加时间窗口、Token、From/To 等条件继续缩小范围。

---

# 何时调用

出现以下任一情况时调用：

- 对方说已经转账，但用户没有到账
- 共享屏幕或钱包 App 显示 Transaction Successful
- 对方要求双方“同步按转账”
- 对方以“冷钱包”“链上确认慢”“交易所入账延迟”为理由催付款
- 对方拒绝提供完整 TxHash / TxID
- 用户手里有 Block Height、时间、地址、金额、TxHash 中的任意信息
- 用户希望从区块高度反推时间，或从时间/地址/金额反查交易
- 用户需要形成一份第三方可复验的链上证据链

---

# 输入 Schema

至少先确定 Chain / Network；如果无法确定，不要跨链猜测。

```yaml
chain: TRON                  # 必填或需先确定
network: mainnet             # 默认主网，但需结合场景确认

tx_hash: null                # 核心字段，可选
from_address: null           # 核心字段，可选
to_address: null             # 核心字段，可选
block_height: null           # 核心字段，可选
timestamp: null              # 核心字段，可选
timezone: "UTC+8"            # 若用户给的是当地时间，必须保留时区

amount: null                 # 强辅助字段
token: null                  # 如 USDT
token_contract: null         # 如已知，优先使用

search_window_minutes: 5     # 初始窗口
only_confirmed: true
```

如果只有金额而没有 Chain / Token / 时间 / 地址等可定位信息，应先索取更多线索，而不是全链盲扫。

---

# Step 0：建立“声称字段表”

先把用户或对方提供的信息原样记录，不做推断：

```text
Claimed TxHash:
Claimed From:
Claimed To:
Claimed Block:
Claimed Time:
Claimed Amount:
Claimed Token:
```

然后另建一组：

```text
Ledger-derived TxHash:
Ledger-derived From:
Ledger-derived To:
Ledger-derived Block:
Ledger-derived Time:
Ledger-derived Amount:
Ledger-derived Token:
```

**永远不要把“对方展示值”和“从主网推导值”混在一起。**

---

# 自动选择最短核验路径

## 路径 A：已有 TxHash

TxHash 是最强单点锚。

执行：

1. 在对应主网查询 TxHash
2. 若存在，读取：
   - Block
   - Timestamp
   - From
   - To
   - Status / Receipt
   - Token Contract
   - Amount / Transfer Event
3. 将所有字段与 claimed values 逐项比较
4. 若 TxHash 不存在，换第二个独立数据源复核一次；仍不存在则输出 **Not Found in verified sources**

TxHash 字符串“格式正确”不等于交易真实存在。

---

## 路径 B：Block Height + Timestamp

这是识别伪造交易页面最快的组合之一。

执行：

1. 查询 Block Height 的主网原始区块头
2. 读取区块 Timestamp
3. 转换为用户指定时区
4. 与 Claimed Timestamp 比较

如果二者差异远超正常出块/显示误差：

> **Block 与 Time 存在硬性矛盾。**

“交易所入账延迟”无法改变已经产生的区块时间。

---

## 路径 C：Timestamp → 真实 Block

当对方给出的 Block 可疑时，不要继续围绕那个 Block 打转。

执行：

1. 根据目标 Timestamp 找接近时间的已知区块
2. 按该链平均出块速度估算高度
3. 查询前后区块 Timestamp
4. 二分/迭代直到找到最接近目标时间的真实 Block
5. 保存：
   - true_block_for_claimed_time
   - true_block_timestamp

**估算只用于定位，最终必须由节点返回的区块 Timestamp 确认。**

---

## 路径 D：Block + 地址 / 金额

当已有真实 Block，再扫描该块：

1. Transactions
2. Receipts
3. Event Logs / Token Transfer Events

检查：

- 目标 TxHash 是否出现
- From / To 是否出现
- Token Contract 是否匹配
- Amount 是否匹配
- Receipt 是否成功

不要只查 transaction body。代币转账可能由智能合约间接触发，因此必须同时检查 Event Logs。

---

## 路径 E：To Address + Timestamp

这是普通用户最容易理解、最适合反诈骗的一条路径。

问题只有一句：

> **在他说已经付款的这段时间，我的收款地址到底有没有收到对应资产？**

执行：

1. 查询 To Address 的入账历史
2. 限定 Token / Contract
3. 只查 confirmed
4. 先查 Claimed Time ±5 分钟
5. 如果没有，再查 ±30 分钟
6. 如仍存在“时间偏差”争议，再查 ±60 分钟
7. 对返回交易逐笔核对 Amount、From、TxHash、Block、Timestamp

如果查询成功但结果为空，准确表述为：

> **在已验证的链、Token、地址和时间窗口内，没有找到符合条件的已确认转入。**

不要无限扩大结论为“这笔交易在任何时间都不存在”。

---

## 路径 F：地址 + Amount

地址 + 金额通常可以缩小范围，但可能不唯一。

执行：

1. 明确 To 还是 From
2. 明确 Token / Contract
3. 优先要求一个大致日期或时间范围
4. 查询该地址在窗口内的 Transfer
5. 筛 Amount
6. 若多笔同额，继续用 From / Block / Time / TxHash 消歧

---

## 路径 G：From + To

执行：

1. 查询双方地址间的交易或 Token Transfer
2. 按时间、Token、Amount 缩小
3. 得到候选 TxHash
4. 再进入路径 A 完整核验

---

# 结果必须形成“证据矩阵”

```markdown
| 字段 | 对方声称 | 主网实际 | 是否一致 |
|---|---|---|---|
| TxHash | ... | ... | ✅/❌/? |
| From | ... | ... | ✅/❌/? |
| To | ... | ... | ✅/❌/? |
| Block | ... | ... | ✅/❌/? |
| Timestamp | ... | ... | ✅/❌/? |
| Amount | ... | ... | ✅/❌/? |
| Token Contract | ... | ... | ✅/❌/? |
| Receipt / Event | ... | ... | ✅/❌/? |
```

---

# 判定状态

只使用以下可解释状态：

### VERIFIED
公开主网可以找到该交易，关键字段一致。

### PARTIALLY_VERIFIED
找到候选交易，但仍有字段缺失或无法确认。

### CONTRADICTORY
至少一个关键字段与公开账本发生明确冲突，例如 Block 对应时间与声称时间明显不符。

### NOT_FOUND_IN_SEARCH_SCOPE
在明确记录的链、地址、Token 和时间窗口内没有找到对应交易。

### INSUFFICIENT_EVIDENCE
现有字段不足以唯一定位，需要更多信息。

避免仅凭一个异常直接输出“诈骗”“伪造 App”等身份或动机结论。可以说：

> **对方展示的链上交易信息无法与公开主网账本对应。**

---

# TRON / TRC20 Adapter

## 1. Block Height → Block + Timestamp

```http
POST https://api.trongrid.io/wallet/getblockbynum
Content-Type: application/json

{"num": <BLOCK_HEIGHT>}
```

读取：

```text
block_header.raw_data.number
block_header.raw_data.timestamp
```

TRON Timestamp 使用 Unix milliseconds。

---

## 2. TxHash → Transaction

```http
POST https://api.trongrid.io/wallet/gettransactionbyid
Content-Type: application/json

{"value": "<TXID>"}
```

同时查 Receipt / Transaction Info：

```http
POST https://api.trongrid.io/wallet/gettransactioninfobyid
Content-Type: application/json

{"value": "<TXID>"}
```

---

## 3. Block → Receipts / Event Logs

```http
POST https://api.trongrid.io/wallet/gettransactioninfobyblocknum
Content-Type: application/json

{"num": <BLOCK_HEIGHT>}
```

TRC20 标准 Transfer Event 的 topic0：

```text
ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
```

需要核对 indexed From / To 与 data 中的 value。

---

## 4. Address + Time → TRC20 inbound history

```http
GET https://api.trongrid.io/v1/accounts/<ADDRESS>/transactions/trc20
```

推荐参数：

```text
only_confirmed=true
only_to=true
contract_address=<TOKEN_CONTRACT>
min_timestamp=<START_MS>
max_timestamp=<END_MS>
order_by=block_timestamp,asc
limit=200
```

如果结果可能超过一页，必须处理分页 / fingerprint；不能只看第一页就宣称完整窗口没有记录。

---

# Amount 处理

必须按 Token decimals 还原。

例如某 Token 为 6 decimals：

```text
raw_value = human_value × 1,000,000
```

不要默认所有 Token 都是 6 位或 18 位；应读取 Token metadata / contract decimals。

---

# AI 交叉评审

AI 可以帮助：

- 解析原始 JSON
- 换算 Timestamp
- 解码 calldata / event value
- 构建候选交易
- 对比证据矩阵
- 生成查询命令

但 **AI 不是证据源**。

如果条件允许，可让两个不同模型分别读取同一批原始链上数据，独立给出：

```text
1. Block ↔ Time 是否一致
2. TxHash 是否存在
3. From / To 是否一致
4. Amount / Token 是否一致
5. Address window 是否存在入账
```

然后比较两份结果。出现分歧时，回到节点原始 JSON，而不是让模型“投票”。

正确表述：

> **多模型交叉评审用于降低解析错误和单模型幻觉风险；最终事实依据仍是任何人都能重复查询的公开链上数据。**

---

# 原始证据保全

若核验涉及诈骗、威胁、争议交易：

1. 保存原始 JSON，不修改
2. 保存查询 URL / curl / 请求参数
3. 保存查询时间与时区
4. 保存完整 TxHash、Block、地址
5. 对原始文件计算 SHA-256
6. 如有原始录屏，保存原文件及 SHA-256
7. 公开发帖时对个人身份、住址、银行账号等敏感信息打码

录屏/截图的用途是证明：

> **对方当时展示了哪些字段。**

公开账本的用途是证明：

> **主网上实际发生了什么。**

两者必须严格区分。

---

# Agent 执行规则

1. 先确定 Chain / Network
2. 原样记录所有 claimed fields
3. 根据现有字段选择最短查询路径
4. 优先官方节点 / 官方 API，再用独立浏览器交叉复核
5. 永远保留原始响应
6. 处理时区、Token decimals、分页
7. 不因为钱包 UI 显示 Success 就视为成功
8. 不因为 TxHash 格式合法就视为存在
9. 不因为两个 AI 结论一致就视为链上事实
10. 最终给出“声称 vs 主网实际”的证据矩阵和可复验步骤

---

# 最小输出模板

```markdown
## On-chain Verification

**Network:**  
**Search scope:**  

### Inputs
- TxHash:
- From:
- To:
- Block:
- Time:
- Amount:
- Token:

### Ledger-derived facts
- TxHash:
- From:
- To:
- Block:
- Time:
- Amount:
- Token:
- Receipt/Event:

### Result
VERIFIED / PARTIALLY_VERIFIED / CONTRADICTORY /
NOT_FOUND_IN_SEARCH_SCOPE / INSUFFICIENT_EVIDENCE

### Reproduce
1.
2.
3.

### Notes
- AI assisted with parsing only.
- Source of truth: public ledger / raw node responses.
```

---

# 一句话原则

> **不要相信别人展示给你的链上。拿到 TxHash、From、To、Block、Time 中的一两个有效锚点，顺着公开账本自己把整笔交易反查出来。**
