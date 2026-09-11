# Introduction 资料汇总：跨链原子交换、隐私与可扩展性

更新日期：2026-08-07（Asia/Shanghai）
数据快照：2026-08-01--2026-08-06 的六个日度样本；市值取 2026-08-06 日值。
用途：给 `Introduction` 批注准备可核查的背景、问题陈述和动机材料。本文件是资料底稿，不是已经写入论文的最终表述。

## 1. 老师的批注实际上要求什么

Introduction 通常需要回答四个问题，而不是简单堆砌区块链名词：

1. **背景与现状**：跨链资产交换为什么需要同时处理不同账本、不同密码学和不同隐私模型。
2. **不可替代性**：为什么隐私币/隐私协议不能只用普通 EVM 合约或跨链桥替代；需要给出费用、隐私属性和部署复杂度的事实依据。
3. **现有缺口**：已有 BTC-XMR 原子交换、P2P DEX 和桥接方案分别解决了什么，仍缺少什么（例如多链适配、脚本/密码学差异和可扩展部署）。
4. **本文动机与贡献**：本文方案针对的具体问题、设计目标和贡献边界是什么。

建议把下面的材料转化为“事实 -> 局限 -> 研究问题”，不要把所有数字都塞进正文。

## 2. 先固定比较口径

“交易费用”不能把公链和联盟链放在同一列直接比较：公链通常按交易或计算量支付原生资产费用；Hyperledger Fabric 没有原生 gas，成本主要是组织成员、节点、带宽、存储和运维的基础设施费用。因此表格应把 Fabric 标成“基础设施成本，不适用单笔币价口径”。

隐私术语也要收窄：Bitcoin 不是脚本为零的链；Monero 的隐私是协议默认属性；Zcash 是透明地址与屏蔽地址并存的选择性隐私；Grin 使用 Mimblewimble 的交易结构。本文若使用 “scriptless blockchains”，应在正文中解释为“目标链不依赖通用智能合约脚本即可完成资产转移/隐私保护”，不要暗示所有目标链都没有脚本。

## 3. 可复现的交易费快照

### 3.1 数据源和计算方法

Coin Metrics Community API 的指标目录：

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

## 4. 为什么隐私是协议属性，而不只是一次转账

Monero 官方将 stealth address、ring signatures 和 RingCT 作为默认交易隐私机制，隐藏发送方、接收方和金额：<https://www.getmonero.org/get-started/what-is-monero/>。这能支撑“隐私保护随普通转账一起发生”的论点。

Zcash 官方实现同时支持透明交易和 shielded transaction，屏蔽交易使用零知识证明保护交易内容：<https://github.com/zcash/zcash#what-is-zcash>。因此写作时应说明 Zcash 的隐私是可选路径，不能概括为所有交易默认屏蔽。

Grin 的 Mimblewimble 交易结构与 cut-through 文档可用于说明“交易压缩/可扩展性”和“隐私”之间的协议级联系：

- <https://docs.grin.mw/about-grin/privacy/>
- <https://docs.grin.mw/about-grin/scalability/>

不建议直接使用 Tornado Cash 的“52 倍 gas”说法：证明验证和存款 gas 会随实现、树深和 EVM 状态变化，用户提供的公式与示例数值也不完全一致。若要保留，应另做可复现实验并报告合约版本、gas trace 和 gas price。

## 5. 已有真实项目：按机制分类

### 5.1 原子交换实现（与本文最相关）

- **Eigenwallet/core**：项目自称 battle-tested 的 Monero-Bitcoin atomic-swap DEX，仓库持续更新；可作为工程实现案例：<https://github.com/eigenwallet/core>。
- **COMIT xmr-btc-swap**：BTC/XMR 原子交换实现：<https://github.com/comit-network/xmr-btc-swap>。
- **BasicSwap**：多币种去中心化交换项目，包含原子交换相关实现和运行文档：<https://github.com/basicswap/basicswap>。

这些项目可用来证明“生产级原子交换并非纯理论”，但不能自动证明安全性、流动性或可扩展性已经解决；论文仍需区分代码活跃度、上线交易量和形式化安全证明。

### 5.2 P2P 托管/仲裁交换（不要误称为原子交换）

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

这不是“隐私资产占全球加密市场的比例”。若论文需要全球总市值比例，必须从同一来源、同一时间戳取得全球总市值，不能把 CoinGecko、Coinbase、Binance 和新闻报道的数字混用。CoinGecko 的 “Privacy Coins/Privacy/Privacy Blockchain” 分类定义也不同，应锁定单一分类并在表注说明。

Zcash 屏蔽池占比、Ironwood/NU6.3 迁移等数据是升级时点数据，引用时必须标注“升级前/升级后”口径，不能与普通流通量快照混写。可参考：<https://www.coindesk.com/tech/2026/07/28/zcash-seals-usd1-7-billion-shielded-pool-as-ironwood-upgrade-activates>。

## 7. 可选的额外动机

### 7.1 为什么不直接使用跨链桥

跨链桥通常引入托管者、验证者集合或消息验证组件，扩大了跨域信任面。Chainlink 的教育材料汇总了桥接漏洞与历史损失，可作为动机线索：<https://chain.link/education-hub/cross-chain-bridge-vulnerabilities>。正文应写成“桥接会引入额外信任与攻击面”，不要只依赖一个累计损失数字，也不要把桥的统计数字当作原子交换安全性证明。

### 7.2 监管和中心化入口收缩

MiCA 法规原文：<https://eur-lex.europa.eu/eli/reg/2023/1114/oj/eng>。交易所下架数量等新闻/商业博客数据只能作为待核查线索；“原子交换是唯一剩余路径”过强，建议改成“原子交换为不依赖中心化托管入口的少数可行路径之一”。

## 8. 来源等级和应避免的坑

### 可直接作为核心证据

Coin Metrics API、Monero/Zcash/Grin/Fabric 官方文档、Zcash ZIP 原文、EUR-Lex、项目官方 GitHub、正式学术论文和 DOI。

### 可作为线索但需二次核验

YCharts、Coinbase、Binance Academy、CoinLaw、Monerica、CryptoNews、AInvest、TradingView、新闻报道和搜索摘要。它们可以帮助定位数据，但在 IEEE 论文中应尽量替换为可复现 API、区块浏览器或原始研究。

### 三个具体风险

1. YCharts 的 “Average Cost Per Transaction” 可能是矿工总收入除以交易数，不等于用户支付的手续费；本文只采用明确的 `FeeTotNtv` 费用指标。
2. 费率、市值、屏蔽池规模都会波动，必须把访问日期和窗口写进表注。
3. Haveno 的多签/仲裁交换、桥接和原子交换的信任模型不同，不能放在同一“原子交换已被证明安全”的句子里。

## 9. 还需要手动补的资料

- 若老师要求“钱包端典型费率”，可在 <https://xmrfees.com/> 或区块浏览器记录一次带时间戳的 Monero 交易，并注明交易大小、优先级和区块高度。
- 若要量化 Fabric 成本，应选择一个云区域、节点规格、存储量和运行时长，再引用 AWS Managed Blockchain 定价页；不能只写“无 gas 所以免费”。
- 若要写支付市场份额，必须获得统计样本、时间范围、分母和原始数据；没有方法说明时不要把单一网站的比例推广为全球市场事实。

## 10. 可直接改写进 Introduction 的逻辑骨架

> Existing cross-chain exchange systems must reconcile heterogeneous transaction formats, cryptographic assumptions, and privacy semantics. Public smart-contract platforms can express complex exchange logic, but privacy protection is commonly supplied by an additional application layer whose execution cost and trust assumptions depend on the contract. In contrast, privacy-oriented ledgers such as Monero and shielded Zcash encode privacy at the protocol level, while consortium systems such as Hyperledger Fabric expose infrastructure rather than per-transaction gas costs. This heterogeneity makes a single script-heavy exchange design difficult to deploy across chains. Existing BTC-XMR atomic-swap implementations demonstrate feasibility, but the ecosystem still lacks a uniformly specified, scalable, and privacy-aware protocol that makes the cross-chain safety and refund conditions explicit. This motivates the design presented in this paper.

这段只是结构模板；正式投稿时应替换为论文自己的引用键，并把“scalable”“uniformly specified”等判断绑定到后文问题定义和实验结果。

## 11. 引言写作检查清单

- [ ] 每个数字都有来源、日期、单位、分母和计算公式。
- [ ] 公链单笔费用与 Fabric 基础设施成本分开叙述。
- [ ] Monero 默认隐私、Zcash 选择性屏蔽、Grin Mimblewimble 的差别写清楚。
- [ ] 原子交换、P2P 多签托管、跨链桥三类机制没有混称。
- [ ] 生产项目只证明“已有实现/可运行”，不替代形式化安全论证。
- [ ] 全球市值比例和隐私资产分类只使用同一来源、同一时间戳。
- [ ] 论文正文只保留能服务于“背景 -> 缺口 -> 本文贡献”的数据。
