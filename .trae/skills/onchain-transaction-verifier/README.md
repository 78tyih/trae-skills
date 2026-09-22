# On-chain Transaction Verifier

一个用于核验链上交易真实性、识别“假转账成功 / 假钱包成功页 / 共享屏幕同步付款”场景的 Trae Skill。

## 核心输入

五个核心字段：

- TxHash / TxID
- From（付款地址）
- To（收款地址）
- Block Height（区块高度）
- Timestamp（交易时间）

辅助字段：

- Amount
- Token / Contract
- Chain / Network

通常拿到一个强锚点（尤其 TxHash）或两项可组合信息，就可以从公开账本反推出其它字段并进行交叉验证；若组合不唯一，则继续加入时间窗口、Token、金额或地址消歧。

## 核心思想

> 区块链是一套公开账本。截图、共享屏幕和钱包 UI 只是对方展示的信息；节点和链上原始数据才是事实源。

Skill 会自动选择最短核验路径，例如：

- TxHash → Block / Time / From / To / Amount
- Block + Time → 验证是否自洽
- Time → 反推真实 Block
- Block + To / Amount → 扫 Transactions + Receipts + Event Logs
- To + Time → 查真实收款地址在指定窗口内是否有入账
- Address + Amount → 筛候选交易，再用其它字段消歧

## 安装

将本目录放在：

```text
.trae/skills/onchain-transaction-verifier/
```

Trae 可通过 `SKILL.md` 识别。

## 示例调用

```text
用 onchain-transaction-verifier 核验这笔交易：

chain: TRON
token: USDT
block_height: 84836779
timestamp: 2026-09-21 22:20:24 UTC+8
to_address: <你的收款地址>
amount: 26666

请从公开主网反查并输出“声称 vs 主网实际”的证据矩阵。
```

完整执行规范见 [SKILL.md](./SKILL.md)。

## 重要说明

AI 只负责查询编排、解析和交叉评审。最终事实依据必须是公开主网、官方节点/API 或可重复验证的链上原始数据。
