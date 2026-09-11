# Introduction 资料汇总：跨链原子交换、隐私与可扩展性

## 1. 要求

1. **背景与现状**：跨链资产交换为什么需要同时处理不同账本、不同密码学和不同隐私模型。
2. **不可替代性**：为什么隐私币/隐私协议不能只用普通 EVM 合约或跨链桥替代；需要给出费用、隐私属性和部署复杂度的事实依据。
3. **现有缺口**：已有 BTC-XMR 原子交换、P2P DEX 和桥接方案分别解决了什么，仍缺少什么（例如多链适配、脚本/密码学差异和可扩展部署）。
4. **本文动机与贡献**：本文方案针对的具体问题、设计目标和贡献边界是什么。


## 2. 先固定比较口径

“交易费用”不能把公链和联盟链放在同一列直接比较：公链通常按交易或计算量支付原生资产费用；Hyperledger Fabric 没有原生 gas，成本主要是组织成员、节点、带宽、存储和运维的基础设施费用。因此表格应把 Fabric 标成“基础设施成本，不适用单笔币价口径”。

Bitcoin 不是脚本为零的链；Monero 的隐私是协议默认属性；Zcash 是透明地址与屏蔽地址并存的选择性隐私；Grin 使用 Mimblewimble 的交易结构。本文若使用 “scriptless blockchains”，应在正文中解释为“目标链不依赖通用智能合约脚本即可完成资产转移/隐私保护”，不要暗示所有目标链都没有脚本。

## 3. 可复现的交易费快照

### 3.1 数据源和计算方法

Coin Metrics Community API 指标目录：

<https://community-api.coinmetrics.io/v4/catalog/metrics?metrics=FeeTotNtv,TxCnt,PriceUSD,CapMrktCurUSD>

原始日度数据查询（查询窗口的结束时间按 API 约定处理）：

<https://community-api.coinmetrics.io/v4/timeseries/asset-metrics?assets=btc,eth,xmr,zec,grin&metrics=FeeTotNtv,TxCnt,PriceUSD,CapMrktCurUSD&frequency=1d&start_time=2026-08-01&end_time=2026-08-06&page_size=1000>

对每条链按交易数加权计算美元/笔平均费用：

```text
FeeUSD_avg = sum(FeeTotNtv_d * PriceUSD_d) / sum(TxCnt_d)
```

其中 `FeeTotNtv_d` 是日总手续费（原生资产），`PriceUSD_d` 是日美元价格，`TxCnt_d` 是日交易数。该口径不是钱包显示的下一块预计费用，也不是矿工总收入除以交易数。

### 3.2 六日加权结果

| 链 | 平均费用（USD/笔） | 费用窗口 | 交易数（窗口合计） | 口径说明 |
|---|---:|---|---:|---|
| Bitcoin | 0.347011 | 2026-08-01--08-06 | 3,891,755 | 链上总手续费/交易数，含不同交易大小 |
| Ethereum L1 | 0.094428 | 同上 | 14,708,777 | 含合约调用，不能等同于 21,000 gas 简单转账 |
| Monero | 0.102083 | 同上 | 163,343 | 费用取决于交易体积与网络状况，不按转账金额计费 |
| Zcash | 0.305782 | 同上 | 21,975 | 实际网络费用；协议费率应同时参考 ZIP-317 |
| Grin | 0.000422 | 同上 | 987 | 样本交易量很小，只能作数量级参考 |
| Hyperledger Fabric | 不适用 | 不适用 | 不适用 | 无原生 gas；比较成员/节点/存储/运维成本 |

上表的数值保留六位小数以便复算，不应被写成永久固定价格。正文可四舍五入为 `$0.35/$0.09/$0.10/$0.31`，并在表注写明访问日期和窗口。

### 3.3 协议费率的官方补充

- Monero 官方说明隐私交易的费用与交易大小及网络拥塞有关：<https://www.getmonero.org/resources/user-guides/make-payment.html>。
- Zcash ZIP-317 规定按 logical action 计费，margin fee 为 5,000 zatoshi，grace actions 为 2；普通交易的最低协议费通常从 10,000 zatoshi（0.0001 ZEC）起，不能再把旧的 1,000-zatoshi 常数当作当前统一费率：<https://zips.z.cash/zip-0317>。
- Ethereum 简单转账常用 21,000 gas 作为基线，但合约调用的 gas 消耗可能显著更高；应引用官方 gas 文档并把 gas 与美元费率分开：<https://ethereum.org/en/developers/docs/gas/>。
- Fabric 官方架构文档说明网络由组织、peer、orderer 和策略组成，没有面向用户的原生货币 gas：<https://hyperledger-fabric.readthedocs.io/en/latest/whatis.html>。

## 4. 为什么隐私是协议属性，非一次转账

Monero 官方将 stealth address、ring signatures 和 RingCT 作为默认交易隐私机制，隐藏发送方、接收方和金额：<https://www.getmonero.org/get-started/what-is-monero/>。这能支撑“隐私保护随普通转账一起发生”的论点。

Zcash 官方实现同时支持透明交易和 shielded transaction，屏蔽交易使用零知识证明保护交易内容：<https://github.com/zcash/zcash#what-is-zcash>。因此写作时应说明 Zcash 的隐私是可选路径，不能概括为所有交易默认屏蔽。

Grin 的 Mimblewimble 交易结构与 cut-through 文档可用于说明“交易压缩/可扩展性”和“隐私”之间的协议级联系：

- <https://docs.grin.mw/about-grin/privacy/>
- <https://docs.grin.mw/about-grin/scalability/>



## 5. 已有真实项目：按机制分类

### 5.1 原子交换实现

- **Eigenwallet/core**：项目自称 battle-tested 的 Monero-Bitcoin atomic-swap DEX，仓库持续更新；可作为工程实现案例：<https://github.com/eigenwallet/core>。
- **COMIT xmr-btc-swap**：BTC/XMR 原子交换实现：<https://github.com/comit-network/xmr-btc-swap>。
- **BasicSwap**：多币种去中心化交换项目，包含原子交换相关实现和运行文档：<https://github.com/basicswap/basicswap>。



### 5.2 P2P 托管/仲裁交换

**Haveno** 是基于 Monero、通过 Tor 提供的非托管 P2P DEX：<https://haveno.exchange/>。它使用多签/仲裁式交易流程，与链上 HTLC/适配器签名原子交换的信任假设不同。它适合用作对照案例，说明现实系统仍存在仲裁、保证金和协议实现风险。

### 5.3 生态与节点实现

- Monero 商户目录（VPN、礼品卡、主机、旅行、游戏等类别）：<https://www.getmonero.org/community/merchants/>。
- Zcash Foundation 的 Zebra 节点实现：<https://github.com/ZcashFoundation/zebra>。
- Grin 核心实现：<https://github.com/mimblewimble/grin>。

## 6. 市值和资产占比：只用同一数据源计算

2026-08-06 Coin Metrics 日值：

| 资产 | 市值（USD） |
|---|---:|
| BTC | 1,288,522,357,554 |
| ETH | 231,705,293,233 |
| XMR | 6,756,177,738 |
| ZEC | 7,605,170,759 |
| GRIN | 3,449,875 |

XMR+ZEC+GRIN 合计约 `$14.365B`。若以 BTC+ETH 作为明确的参考分母，则占比约 `0.945%`：

```text
CapShare = (Cap_XMR + Cap_ZEC + Cap_GRIN) / (Cap_BTC + Cap_ETH)
```



Zcash 屏蔽池占比、Ironwood/NU6.3 迁移等数据是升级时点数据，可参考：<https://www.coindesk.com/tech/2026/07/28/zcash-seals-usd1-7-billion-shielded-pool-as-ironwood-upgrade-activates>。

## 7. 可选

### 7.1 为什么不直接使用跨链桥

跨链桥通常引入托管者、验证者集合或消息验证组件，扩大了跨域信任面。Chainlink 的教育材料汇总了桥接漏洞与历史损失，可作为动机线索：<https://chain.link/education-hub/cross-chain-bridge-vulnerabilities>。

### 7.2 监管和中心化入口收缩

MiCA 法规原文：<https://eur-lex.europa.eu/eli/reg/2023/1114/oj/eng>。交易所下架数量等新闻/商业博客数据只能作为待核查线索；“原子交换是唯一剩余路径”过强，建议改成“原子交换为不依赖中心化托管入口的少数可行路径之一”。



