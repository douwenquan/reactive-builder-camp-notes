# 用例拆解卡：个人钱包大额转出监控

> 任务 20 打卡笔记｜学习用例（不是最终项目）

## 1. 监听哪条链、哪个合约的什么事件？

- **源链**：Ethereum Sepolia（chainId `11155111`）
- **合约**：一个 ERC-20 测试代币（例如 SENT）
- **事件**：`Transfer(address indexed from, address indexed to, uint256 value)`
- **过滤条件**：`topic0 = keccak256("Transfer(address,address,uint256)")`，`topic1 = 被保护钱包地址`，即只监听「从我的钱包转出」的转账

## 2. 满足什么条件时触发？

- 单笔转出金额 `value >= threshold`，本练习取 **100 SENT**
- 只处理指定代币、指定钱包的转出；其他事件直接 `return` 过滤
- MVP 加一个 `remaining` 次数上限，防止测试币被耗尽

## 3. 在哪条链做什么动作？

- **目标链**：Ethereum Sepolia
- **动作**：调用 `AlertRecorder.record(...)`，记录 `token / from / to / value / originTxHash / blockNumber`，并 emit `AlertRaised` 事件
- 后续可以接 Telegram / 邮件通知，或做成前端看板

## 4. 如果不用 Reactive，要怎么做？Reactive 让它简单在哪里，代价是什么？

**不用 Reactive：**
- 需要自己写一个监听服务，通过 WebSocket/RPC 订阅 Sepolia 的 `Transfer` 日志
- 需要 7×24 跑服务器、处理断线重连、区块重组、私钥管理和交易发送
- 需要自己实现「检测到事件 → 构造目标链交易 → 重试」的完整逻辑，任何一环挂了都可能漏报

**Reactive 的简化：**
- 不用跑后台服务：订阅和触发逻辑都写在 Reactive 合约里，网络自动监听并调用 `react()`
- 不用自己管理交易发送：`SYSTEM.requestCallbackV_1_0` 会把动作投递到目标链
- 链上可验证：源链事件、Reactive 的 `react()`、目标链回调三笔交易都能在浏览器查到

**代价：**
- 订阅只支持等值匹配，不能写「金额大于 X」这类条件，复杂判断只能在 `react()` 里做
- 每次 `react()` 都要消耗 lREACT；不管是否满足条件都会收费，`revert` 也收费
- 需要持续维护 Reactive 合约的 lREACT 余额，`debt > 0` 会静默停止
- 回调合约在目标链要付 ETH gas；不适合做复杂链下计算/外部 API 调用

## 5. 简图

```mermaid
flowchart LR
    A[Sepolia<br/>ERC-20 测试代币] -->|Transfer 事件<br/>from = 我的钱包| B[Reactive Lasna<br/>Reactive 监控合约.react()]
    B -->|value >= 100 SENT| C[Sepolia<br/>AlertRecorder.record()]
    C --> D[AlertRaised 事件<br/>链上可查]
```

## 6. 结论

这个场景适合用 Reactive：事件驱动、规则简单、需要跨链动作、个人自动化。它不需要复杂的链下服务，正好是 Track 2「Set It and Forget It」的典型用例。
