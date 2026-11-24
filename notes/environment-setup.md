# 环境准备：测试钱包 + Foundry + lREACT

> 任务 10 打卡笔记
> 日期：2026-11-24（北京时间）

## 1. 测试钱包地址

```
0x659DE9e47f3F236207c0C8b964e0D6893742bE42
```

> 仅用于本期测试网。私钥不上传、不截图、不提交到仓库。

## 2. lREACT 余额（Lasna）

- 余额：**5.25 lREACT**
- Reactscan：https://lasna.reactscan.net/address/0x659DE9e47f3F236207c0C8b964e0D6893742bE42
- 换取方式：向 Ethereum Sepolia 上的官方 lREACT faucet 合约转入 0.05 ETH（1 ETH → 100 lREACT，单笔不超过 5 ETH）

## 3. Sepolia ETH 余额

- 余额：**0.108617532033427564 ETH**
- 用于源链事件交易和回调合约 gas。

## 4. Foundry 版本

```text
$ forge --version
forge Version: 1.4.2-stable
Commit SHA: 828441d243f552f82c5a89bdc818e52bdd57b26b
Build Timestamp: 2025-10-18T08:19:39.947006000Z (1760775579)
Build Profile: maxperf

$ cast --version
cast Version: 1.4.2-stable
Commit SHA: 828441d243f552f82c5a89bdc818e52bdd57b26b
Build Timestamp: 2025-10-18T08:19:39.947006000Z (1760775579)
Build Profile: maxperf
```

截图：`assets/forge-version.png`

## 5. 学习仓库

https://github.com/douwenquan/reactive-builder-camp-notes

## 6. 余额截图

`assets/wallet-balances.png`

## 完成标准自查

- [x] `forge --version` 有输出
- [x] 有可用的测试钱包地址
- [x] Sepolia ETH 足够（0.1086 ETH）
- [x] Lasna lREACT > 0（5.25 lREACT）
- [x] 已保存余额截图 / Reactscan 链接
- [x] 已保存 `forge --version` 截图
