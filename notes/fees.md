# 读懂费用：Lasna 系统合约参数

> 任务 30 打卡笔记 · 查询时间 2026-11-23（北京时间） · 网络：Reactive Lasna（chainId 5318007）

## 一、三条 cast 命令与输出

系统合约 `SYSTEM`：`0x8888888888888888888888888888888888888888`
RPC：`https://lasna-rpc.rnk.dev/`

```bash
cast call 0x8888888888888888888888888888888888888888 "_gasPriceCoeffPer1000()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 输出：1000

cast call 0x8888888888888888888888888888888888888888 "_extraGas()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 输出：100000

cast call 0x8888888888888888888888888888888888888888 "_maxChargeGas()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 输出：50000
```

参考：Reactive 官方文档 Economy & System Interactions — https://dev.reactive.network/economy

## 二、一次 react() 的费用怎么算？

官方公式：

```
fee = p_base × C × (g_used + K)
```

- `p_base`：`max(tx.gasprice, block.basefee)`，即实际基础 gas 价格
- `C`：`_gasPriceCoeffPer1000() / 1000`；Lasna 上该值为 `1000`，所以 **C = 1**
- `g_used`：react() 实际消耗的 gas
- `K`：`_extraGas()`，Lasna 上为 **100,000 gas** 的固定附加费

带入 Lasna 的参数：

```
fee = p_base × 1 × (g_used + 100000)
```

`_maxChargeGas()` 在 Lasna 上是 **50,000**，它不直接进入费用公式，而是限制系统合约在 `pay()` 结算时最多可用的 gas；如果自定义 `pay()` 逻辑超过这个预算，会支付失败并留下 debt。

## 三、balance、reserves、debt 分别是什么？为什么有一点 debt 就停止执行？

| 概念 | 含义 | 后果 |
|---|---|---|
| **balance** | 合约地址自身的余额（合约可支配） | 需要合约主动 `pay()` / `coverDebt()` 才能用于费用 |
| **reserves** | 预付在系统合约里的额度 | 收费时优先从 reserves 扣；`depositTo()` 可直接为任意合约充值 |
| **debt** | 已发生但未支付的欠款 | **只要 debt > 0，系统合约不会调用 react()，回调 Proxy 也不会投递回调——合约静默停止** |

为什么 debt > 0 就停止：系统合约的 `trigger()` 生命周期里，**第一步就是检查 debt**，在消耗任何 gas 之前就判断；有欠款直接跳过，不执行也不报错，所以表现是「突然什么都不触发」。这是设计上的保护：避免合约继续产生无法支付的费用。恢复方式：
- 给合约转 lREACT（Reactive 侧）后调用 `coverDebt()`；
- 或用 `depositTo(address)` 从外部直接为合约充值；
- 回调合约则用目标链原生币（ETH）充值给 Callback Proxy。

## 四、react() 执行回滚了还收费吗？对合约设计意味着什么？

**收费。** 官方文档明确：react() revert 会被系统合约捕获，记录为 `ReactiveContractReverted` 事件，然后**照样按实际消耗的 gas 收费**。Callback 失败同样收费（`CallbackFailure`）。

对设计的启示：
1. **不要用 revert 过滤事件**：不满足条件时用 `return` 提前退出，避免白白付费。
2. **尽量把过滤放在订阅层**：订阅只支持等值匹配，能确定的 chainId / 合约地址 / topic 都写具体，减少无关事件进入 react()。
3. **给自动化设次数 / 金额上限**：例如 `remaining` 计数，归零后 `unsubscribe()`，防止测试币被持续扣光。
4. **持续监控 debt 和余额**：react() 越复杂、调用越频繁，成本越高；上线前先用 `cast estimate` / 测试网实测。
5. 对于高频场景（如订阅全部代币的 Transfer），先用 `_limit` 控制处理次数，或改成监听特定代币。

## 五、一句话总结

Lasna 当前：`_gasPriceCoeffPer1000 = 1000`（系数 1）、`_extraGas = 100,000`、`_maxChargeGas = 50,000`；费用 = gas 价格 × (实际 gas + 100,000)；balance/reserves/debt 三本账里，**debt 是最危险的——大于 0 就直接静默停机**；react() 回滚也收费，所以过滤用 `return`，并给自动化设上限。


## 六、费用思考题

如果 `_extraGas` 从 100,000 变成 200,000，其他条件不变，费用约变成原来的多少倍？

公式：

```
旧 fee = p_base × (g_used + 100,000)
新 fee = p_base × (g_used + 200,000)
倍数 = (g_used + 200,000) / (g_used + 100,000)
```

- 若 `g_used ≈ 100,000`：倍数 = 300,000 / 200,000 = **1.5 倍**
- 若 `g_used ≈ 50,000`：倍数 = 250,000 / 150,000 ≈ **1.67 倍**
- `g_used` 越大，倍数越接近 1

关键点：**不是 2 倍**，因为固定附加 gas 只是总费用的一部分。
