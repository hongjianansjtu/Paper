# 基于公证人协调与哈希时间锁的跨链资产交换系统设计与实现

## 摘要

随着区块链生态的持续发展，单一区块链系统已经难以满足多链资产流通、跨平台业务协作和异构链互操作的需求。不同区块链在账户模型、共识机制、智能合约能力、交易确认方式以及脚本表达能力方面存在明显差异，使得资产无法像同一系统内部转账一样直接从一条链迁移到另一条链。跨链资产交换因此成为区块链互操作研究中的重要问题。跨链协议不仅需要完成源链与目标链之间的状态协调，还需要保证交易过程中的原子性、安全性和失败回滚能力，避免出现一方资产已经锁定或支付，而另一方无法获得对应资产的情况。

针对上述问题，本文设计并实现了一个基于公证人协调与哈希时间锁的跨链资产交换原型系统。系统采用 Alice - ManagerA - Notary - ManagerB - Bob 的角色协作模型，其中 Alice 为跨链发起方，Bob 为目标链接收方，ManagerA 与 ManagerB 分别负责源链和目标链上的资产池管理，Notary 作为公证与中继节点负责监听链上事件并协调目标链响应。在支持智能合约的链上，系统通过 HTLC 哈希时间锁机制实现资产锁定、条件领取和超时退款；在 BTC 场景中，虽然 Bitcoin Script 本身支持哈希锁和时间锁操作码，能够实现基本的 HTLC，但链上会暴露条件脚本且 UTXO 模型不适合维护复杂跨链状态，因此系统引入 Schnorr adaptor signature，将锁定条件转化为签名补全条件，使链上交易表现为普通签名交易，提升隐私性和链上简洁性。本文围绕系统架构、核心机制、合约实现、公证人协调流程、scriptless 适配方式、安全性分析以及实验展示进行论述，证明该原型系统能够在统一跨链流程下兼容不同执行能力的区块链，为异构链之间的资产交换提供一种可行的设计思路。

**关键词：** 跨链资产交换；哈希时间锁；公证人机制；Schnorr Adaptor Signature；异构区块链；智能合约

## 第一章 绪论

### 1.1 研究背景：跨链资产交换

区块链技术通过分布式账本、密码学签名和共识机制实现了去中心化的价值记录与转移。随着以太坊、比特币、BSC、Conflux 等公链以及 Hyperledger Fabric 等联盟链的发展，区块链逐渐形成公链与联盟链并存的多链生态格局。不同链往往面向不同的应用场景，有的链侧重通用智能合约执行，有的链侧重高性能交易处理，有的链则以安全性和价值存储为核心目标。然而，多链生态也带来了资产和状态割裂的问题。某一条链上的账户余额、交易记录和合约状态通常不能被另一条链直接读取或修改，用户如果希望在不同链之间完成资产交换，不能简单地调用一次普通转账交易，而必须借助跨链协议完成多链状态协调。

跨链资产交换的本质并不是把同一个资产实体从一条链物理搬运到另一条链，而是在两个相互独立的账本系统之间建立可信的价值对应关系。以 Alice 与 Bob 的交换为例，Alice 希望在源链上支付一定数量的资产，Bob 在目标链上收到等值资产。由于两条链的交易确认相互独立，如果没有额外机制，Alice 支付后 Bob 可能无法收到资产，或者 Bob 收到资产后 Alice 的支付无法完成。跨链系统必须解决”谁先执行””如何证明对方已执行””失败后如何回滚”等核心问题。

在传统中心化交易平台中，这类问题通常由平台托管资产并撮合交易解决，但这种方式引入了中心化信任风险。平台一旦出现作恶、被攻击、运营失败或提现限制，用户资产可能遭受损失。去中心化跨链交换希望通过链上合约、密码学锁定、事件监听和中继机制降低对单一中心化实体的依赖，使资产释放条件由可验证的协议规则控制。本文所研究的系统正是面向这一问题的原型实现，旨在通过多角色协作和密码学条件锁定，为跨链资产交换提供一种结构清晰、安全可分析的设计方案。

### 1.2 研究意义

跨链资产交换的研究具有较强的现实意义和工程价值。从需求侧看，多链生态已经成为区块链应用发展的常态，用户希望将一条链上的原生资产转换为另一条链上的资产，或者在不同链之间使用同一套应用服务。如果缺少跨链能力，各链将形成相互割裂的价值孤岛，限制区块链应用的扩展和用户体验的统一。

从技术方案看，现有跨链方案大致可分为中继链验证、侧链/桥托管和原子交换三类。中继链验证安全性较强但实现成本高，对链的合约能力和验证能力要求较高；侧链/桥托管实现相对简单但引入了中心化信任假设，削弱了区块链系统本身强调的去中心化特性。基于 HTLC 的原子交换提供了一种较轻量的折中方案，通过 hashLock 和 timeLock 将两侧资产的领取条件绑定在同一个 secret 上，能够在不完全信任对手方的情况下完成交换，同时结合公证人协调机制可以兼顾流程自动化与安全性。

从技术难点看，异构链兼容是跨链系统必须面对的挑战。EVM 类链可以直接部署智能合约，通过合约状态记录 hashLock、timeLock、swapId 和资产锁定情况；BTC 虽然通过 Script 支持 OP_SHA256 和 OP_CHECKLOCKTIMEVERIFY 等操作码，能够实现基本的哈希时间锁，但其 UTXO 模型不具备持久化状态管理能力，链上 HTLC 脚本也会暴露条件分支，降低交易隐私性。如果跨链系统只支持一种合约模型，就难以适应不同链的执行能力和隐私需求差异。本文将 HTLC 与 Schnorr adaptor signature 放在统一跨链架构下讨论：对于合约能力较强的链使用链上 HTLC，对于 BTC 场景则将条件锁定逻辑放入签名协议中，使链上交易表现为普通转账，从而在统一流程下兼顾不同链的特性与隐私要求。

从工程实践角度看，本文工作不仅具有理论分析价值，也具有系统实现价值。项目中包含 Solidity 智能合约、Python 后端协调服务、BTC adaptor signature 模拟模块和前端可视化页面，能够较完整地展示跨链资产交换从协议设计到代码实现再到流程演示的全过程。

### 1.3 本文主要工作

本文围绕跨链资产交换原型系统，从架构设计、核心机制实现和安全性分析三个层面展开工作。

在架构设计层面，本文提出了 Alice - ManagerA - Notary - ManagerB - Bob 的跨链角色模型。该模型将用户、资金池管理者、公证中继节点和目标链接收方明确区分，使系统能够清晰表达资产控制边界与消息协调之间的关系，避免将跨链流程简单等同于单个合约调用。系统将跨链资产交换拆分为用户发起、资金池锁定、事件监听、目标链响应、秘密揭示、资产释放与超时退款等阶段，每个阶段由对应角色驱动。

在核心机制实现层面，本文针对不同链的执行能力设计了两种条件锁定方式。对于支持智能合约的链，系统通过 HTLC 实现资产锁定，使用 hashLock 约束领取条件、timeLock 约束退款条件、swapId 关联同一笔跨链请求，形成源链与目标链的状态配对。对于 BTC 等链，虽然 Bitcoin Script 支持基本的哈希时间锁，但 UTXO 模型不适合维护复杂跨链状态且链上脚本会暴露条件逻辑，因此系统引入 Schnorr adaptor signature，将完成条件嵌入签名补全过程，以密码学方式实现条件化交易，同时保持链上交易的简洁性和隐私性。在两种机制之上，Notary 作为公证人监听源链合约事件，读取跨链参数并协调目标链 Manager 创建对应锁定，承担跨链消息传递和流程推动的职责。工程实现方面，智能合约模块负责资产池和 HTLC 状态管理，Python 后端模块负责跨链协调，前端模块用于展示跨链流程中各角色状态、密码学参数和交易进度的变化。

在安全性分析层面，本文对系统的原子性、可退款性、Notary 信任边界、Manager 约束和异构兼容能力进行了讨论，说明该系统在原型层面能够满足跨链资产交换的基本安全要求。

本文其余部分组织如下：第二章介绍 HTLC 和 Schnorr adaptor signature 的相关技术背景；第三章阐述系统总体设计，包括角色模型、模块划分和跨链流程；第四章详细描述核心机制的设计与实现；第五章进行系统安全性分析；第六章展示系统实现与实验验证；第七章总结全文并讨论后续工作方向。

## 第二章 相关技术

### 2.1 HTLC 哈希时间锁

HTLC 全称为 Hashed TimeLock Contract，即哈希时间锁合约。它是跨链原子交换中常用的一种机制，其核心思想是将资产领取条件拆分为两个部分：哈希锁和时间锁。哈希锁要求领取方必须提交一个 secret，使得该 secret 经过哈希计算后等于合约中预先保存的 hashLock；时间锁则规定如果在指定时间内没有人提交正确 secret，资产可以被原始锁定方取回。

HTLC 的基本流程可以描述为：Alice 首先生成随机 secret，并计算 hashLock = H(secret)。在发起跨链交换时，Alice 并不公开 secret，而是将 hashLock 写入源链锁定合约。目标链也使用同一个 hashLock 创建对应锁定。这样，两条链上的资产虽然位于不同账本中，但它们的领取条件都被同一个 secret 绑定。当某一方为了领取目标链资产而公开 secret 时，另一方也可以使用该 secret 去完成源链或目标链上的对应操作。若 secret 始终没有公开，则双方都无法领取对方资产，最终只能等待 timeLock 到期后退款。

HTLC 的优势在于结构清晰、实现相对简单，并且适合部署在具有智能合约能力的链上。在 Solidity 合约中，hashLock 可以作为 bytes32 类型保存，timeLock 可以用区块高度或时间戳表示。完成交易时，合约只需验证 keccak256(secret) 是否等于 hashLock；退款时，合约只需判断当前区块高度是否超过 timeLock。通过这种方式，链上合约能够自动执行“正确 secret 领取”和“超时退款”两类规则。

不过，HTLC 也存在适用边界。它要求参与链能够表达相应的锁定和退款逻辑，并要求参与方在超时前后及时发起领取或退款交易。值得注意的是，BTC 通过 OP_SHA256、OP_HASH160 和 OP_CHECKLOCKTIMEVERIFY 等操作码，能够在 Script 层面实现基本的 HTLC，Lightning Network 正是建立在这一能力之上。然而，BTC 的 UTXO 模型不具备 EVM 式的持久化状态管理能力，无法像智能合约那样维护 swapId 映射表和资产池状态；同时，链上 HTLC 脚本会显式暴露哈希锁和时间锁条件分支，降低交易隐私性。因此，在面向 BTC 的跨链场景时，可以考虑使用 scriptless 方式将条件逻辑转移到签名协议层，使链上交易表现为普通签名交易，既简化链上脚本复杂度，又提升隐私保护。

### 2.2 Schnorr 签名与 Adaptor Signature

#### 2.2.1 Schnorr 签名基础

Schnorr 签名是一种基于离散对数问题的数字签名方案，其核心特点是结构简洁且具有线性性质。设椭圆曲线群的生成元为 G，签名者的私钥为 x，对应公钥为 P = xG。对消息 m 的签名过程如下：签名者选取随机数 k，计算承诺点 R = kG，然后计算挑战值 e = H(R, P, m)，最后计算签名标量 s = k + e·x。最终签名为 (R, s)。

验证者收到签名 (R, s) 后，计算 e = H(R, P, m)，并检验等式 sG = R + e·P 是否成立。由于 sG = (k + e·x)G = kG + e·xG = R + e·P，合法签名必然通过验证。

Schnorr 签名的线性性质是 adaptor signature 的数学基础。由于签名标量 s 是随机数 k 与私钥 x 的线性组合，可以在 s 上叠加或拆分标量值而不破坏签名的代数结构。这一性质使得”将秘密值嵌入签名补全过程”成为可能。

#### 2.2.2 Adaptor Signature 构造

Adaptor signature（适配签名）在 Schnorr 签名的基础上引入一个额外的秘密点 T = tG，其中 t 是待传递的秘密标量。适配签名协议包含四个阶段：预签名（PreSign）、预验证（PreVerify）、补全（Adapt）和提取（Extract）。

PreSign 阶段：签名者选取随机数 k，计算调整后的承诺点 R' = kG + T，然后计算挑战值 e = H(R', P, m) 和预签名标量 s' = k + e·x。输出预签名 (R', s')。注意此时 s' 并不是相对于 R' 的有效 Schnorr 签名，因为 s'G = kG + e·P = R' - T + e·P，验证等式 s'G = R' + e·P 不成立。

PreVerify 阶段：验证者已知公开的秘密点 T，可以检验 s'G = R' - T + e·P 是否成立。该等式验证了预签名的正确性——即确认签名者确实使用了与 T 对应的承诺偏移，且签名者知道私钥 x。

Adapt 阶段：持有秘密 t 的一方计算完整签名标量 s = s' + t。此时 sG = s'G + tG = (R' - T + e·P) + T = R' + e·P，验证等式成立，(R', s) 构成对消息 m 的有效 Schnorr 签名。该签名可以被广播上链，完成交易。

Extract 阶段：另一方观察到链上出现的有效签名 s 和此前收到的预签名 s'，通过计算 t = s - s' 即可提取秘密值。由于 s 和 s' 都是公开可见的标量，秘密提取是确定性的。

#### 2.2.3 密码学性质

Adaptor signature 具有以下关键性质，使其适用于原子交换场景：

完整性：如果预签名正确生成且秘密 t 正确补入，补全后的签名必然是有效的 Schnorr 签名，能够通过链上验证。

不可伪造性：不知道私钥 x 的攻击者无法生成能通过 PreVerify 的预签名，不知道秘密 t 的攻击者无法将预签名补全为有效签名。

秘密提取的必然性：一旦补全后的签名被广播上链，任何持有预签名的观察者都能通过 s - s' 提取秘密 t。交易完成与秘密公开之间存在密码学上的必然绑定——不可能在不暴露 t 的情况下使用补全后的签名。

这三条性质共同保证了：在跨链交换中，一方完成交易（广播有效签名）必然导致秘密被另一方获得，从而使另一方也具备完成其侧交易的条件。这与 HTLC 中”公开 secret 领取资产”的逻辑等价，但条件绑定发生在签名协议层而非链上脚本层，链上只呈现一笔普通 Schnorr 签名交易。

## 第三章 系统总体设计

### 3.1 系统角色模型

本文系统采用 Alice - ManagerA - Notary - ManagerB - Bob 的跨链协作模型。该模型将跨链交易中的资金流、消息流和控制流拆分为不同角色，从而提高各环节的可分析性。

Alice 是跨链交易发起方。她负责生成 secret，计算 hashLock，并在源链发起跨链请求。Alice 在源链上锁定资产或中间代币后，等待目标链完成对应资产释放。Bob 是目标链上的接收方，最终应在目标链收到对应资产。在最简单的跨链场景中，Alice 和 Bob 可以是两个不同用户，也可以是同一用户在不同链上的地址。

ManagerA 和 ManagerB 是两侧资产池或跨链 TokenPool 的管理者。ManagerA 位于源链一侧，负责接收、兑换或锁定 Alice 的资产；ManagerB 位于目标链一侧，负责为 Bob 创建对应锁定并在满足条件时释放资产。Manager 并不是任意转移用户资金的中心化操作者，其行为受到合约状态、hashLock、timeLock 和权限控制约束。

Notary 是公证与中继节点，负责监听源链事件并协调目标链操作。当 Alice 在源链发起跨链请求后，源链合约会产生事件，Notary 从事件中读取 swapId、hashLock、amount、recipient 等参数，并将其转发给目标链 ManagerB。Notary 的角色重点在于跨链消息传递和流程协调，而不是最终托管资产。

这种角色模型的优点在于结构清晰。Alice 和 Bob 是用户端，Manager 处理资金池，Notary 处理消息协调，HTLC 或 adaptor signature 处理条件锁定。论文后续章节中的安全性和兼容性分析也都围绕这一角色模型展开。

### 3.2 系统模块划分

从工程实现角度看，系统可以划分为智能合约模块、后端协调模块、BTC/scriptless 模块和前端可视化模块。

智能合约模块主要对应 CrossChainTokenPool 合约。该模块实现跨链资产池、原生币与中间代币兑换、跨链请求发起、Manager 响应、资产领取和超时退款等功能。合约中保存 CrossChainSwap 和 ManagerLock 两类状态，分别表示用户侧锁定和 Manager 侧锁定。二者通过 hashLock 和 swapId 建立关联。

后端协调模块主要由 Python 服务和跨链 coordinator 组成。该模块负责提供 API、连接不同链节点、监听合约事件、处理跨链状态、构造目标链交易并协调 Manager 响应。后端并不替代链上合约的安全判断，而是将不同链之间的消息传递和交易调用组织起来。

BTC 适配模块主要用于展示 Schnorr adaptor signature 机制。该模块通过模拟钱包、区块链和签名流程，说明如何创建 adaptor signature、如何补全签名、如何从完整签名中提取 secret，以及如何将该机制用于跨链条件交换。相比直接在 BTC 上使用链上 HTLC 脚本，adaptor signature 使链上交易表现为普通签名交易，避免暴露条件逻辑。

前端可视化模块用于展示系统流程。它包括跨链交换入口原型和 HTLC 流程演示页面。演示页面能够展示 Alice、ManagerA、Notary、ManagerB、Bob 等角色，展示 secret、hashLock、timeLock、swapId、余额变化和交易阶段，有助于理解跨链协议的执行过程。

### 3.3 总体跨链流程

系统总体流程可以分为正常完成路径和超时退款路径。

在正常完成路径中，Alice 首先生成 secret，并计算 hashLock = H(secret)。随后 Alice 在源链发起跨链请求，提交 hashLock、目标接收地址、金额和 timeLock 等参数。源链 ManagerA 或 TokenPool 合约将 Alice 的资产锁定，并产生 CrossChainInitiated 事件。Notary 监听到该事件后，读取跨链参数，并通知目标链 ManagerB 创建对应的 ManagerLock。ManagerB 在目标链锁定等值资产，等待 Bob 或相关执行方提交 secret。当 secret 被公开后，目标链合约验证 H(secret) 是否等于 hashLock。验证通过后，目标链释放资产给 Bob。由于 secret 已公开，源链也可以用同一 secret 完成结算。至此，一笔跨链交换完成。

在超时退款路径中，如果 Alice 没有公开 secret，或者 Notary 未能及时协调目标链响应，或者 Bob 没有在规定时间内领取资产，则系统进入失败处理。timeLock 到期后，源链上的 Alice 可以调用退款逻辑取回被锁定资产；目标链 ManagerB 也可以对未完成的 ManagerLock 进行退款或回收。这样即使跨链流程中断，资产也不会永久停留在锁定状态。

系统中 swapId、hashLock 和 timeLock 分别承担不同作用。swapId 是跨链请求编号，用于区分不同交换；hashLock 是两侧锁定的公共密码学锚点；timeLock 是失败退出机制。三者共同构成跨链状态机的关键上下文。

上述流程描述的是合约链（HTLC）路径。在 BTC/adaptor signature 路径下，总体角色协作和阶段划分保持一致，但具体的条件锁定方式有所不同：源链或目标链的资产锁定不通过链上 HTLC 脚本表达，而是通过预签名（PreSign）绑定秘密点 T；资产释放不通过链上提交 secret 触发，而是通过持有秘密 t 的一方补全签名并广播交易完成；秘密传递不通过链上 secret 公开，而是另一方从链上出现的有效签名与预签名的差值中提取 t。失败退出机制则通过预设的 timeLock 交易或多签回退路径实现。两种路径在架构层共享同一套角色模型和协调流程，差异仅在于底层条件锁定的实现层。

## 第四章 核心机制设计与实现

### 4.1 合约链上的 HTLC 实现

在支持智能合约的链上，本文系统使用 HTLC 实现跨链资产锁定。合约中的 CrossChainSwap 结构记录用户侧交换状态，包括 hashLock、timeLock、sender、recipient、amount、completed、refunded、createdAt 和 isManagerLocked 等字段。ManagerLock 结构记录 Manager 侧响应锁定状态，包括 hashLock、timeLock、recipient、amount、completed、refunded 和 createdAt 等字段。

当 Alice 调用 initiateCrossChain 发起交换时，合约首先检查 hashLock 是否为空、recipient 是否有效、amount 是否大于零以及用户余额是否足够。随后合约生成 swapId，将 Alice 的资产转入合约地址锁定，并保存 CrossChainSwap 状态。该过程完成后，合约触发 CrossChainInitiated 事件，供 Notary 监听。

目标链 Manager 在收到 Notary 转发的请求后，调用 managerRespondToCrossChain 创建 ManagerLock。该函数要求调用者必须是 Manager，并要求 hashLock 未被重复使用。Manager 侧的锁定通常应设置比用户侧更短的 timeLock，这样可以减少目标链资产长时间占用的风险，也有利于失败时按合理顺序退款。

完成交易时，completeCrossChain 或 completeManagerLock 函数会要求调用方提交 secret。合约计算 keccak256(secret)，并与保存的 hashLock 比较。如果匹配，则将状态标记为 completed，并执行资产释放、销毁或结算逻辑。如果不匹配，交易会回滚。这一机制保证不知道 secret 的参与方无法领取资产。

退款时，refundCrossChain 和 refundManagerLock 依赖 timeLock 判断。只有超过规定时间且交易未完成、未退款时，退款逻辑才能执行。用户侧退款将锁定资产退回 Alice，Manager 侧退款则释放或回收 Manager 锁定的资产。通过完成和退款两条路径，合约形成了一个简明的跨链状态机。

### 4.2 公证人协调流程

公证人协调流程是连接源链和目标链的关键。由于不同链之间不能直接调用彼此合约，源链上的事件不会自动触发目标链状态变化，因此需要 Notary 监听源链事件并发起目标链交易。

在本文系统中，Notary 的典型流程包括：首先连接源链节点，订阅或轮询 CrossChainInitiated 事件；其次解析事件中的 swapId、hashLock、sender、recipient、amount 和 timeLock 等参数；然后根据 targetChainId 或系统配置确定目标链；最后调用目标链 Manager 或 TokenPool 合约创建对应锁定。Notary 还可以记录跨链状态，便于前端查询交易进度。

Notary 的设计需要注意信任边界。它可以延迟消息、漏转发消息或错误转发参数，但它不应拥有绕过 hashLock 直接释放资产的能力。目标链 ManagerLock 的完成仍然依赖正确 secret，源链退款仍然依赖 timeLock。因此，Notary 是流程协调者，不是最终安全性的唯一来源。安全性应由合约条件、签名协议和超时机制共同保证。

### 4.3 BTC 场景中的 adaptor signature

对于 BTC 场景，虽然 Bitcoin Script 本身支持通过 OP_SHA256 和 OP_CHECKLOCKTIMEVERIFY 实现基本的 HTLC，但链上 HTLC 脚本会暴露条件分支、降低隐私性，且 UTXO 模型不适合维护复杂的跨链状态映射。因此，本文系统引入 Schnorr adaptor signature 作为替代方案进行条件化签名。其核心过程是：参与方先构造一份与秘密值相关的预签名，该预签名本身不是有效签名，无法直接完成交易；当秘密值被揭示或补入后，预签名可以转化为正式签名，交易得以完成；同时，另一方可以根据预签名和正式签名之间的差异提取秘密。

这种设计与 HTLC 的目标一致，都是将两侧交易绑定到同一个秘密上。但二者的实现层不同。HTLC 在链上保存 hashLock，由合约验证 secret；adaptor signature 在签名协议中嵌入秘密条件，由签名补全关系保证交易完成与秘密公开之间的联系。

在本文原型中，BTC 模块主要承担机制验证作用。它模拟了 Alice、Bob 等参与方如何创建 adaptor signature，如何在链上观察签名完成，如何提取 secret，以及如何使用该 secret 完成另一侧交换。该模块说明，相比在 BTC 上直接使用链上 HTLC 脚本，adaptor signature 能够在保持相同原子交换语义的同时，避免链上暴露条件逻辑，提供更好的隐私性和链上简洁性。

### 4.4 HTLC 与 adaptor signature 的统一关系

HTLC 与 adaptor signature 并不是两套彼此割裂的跨链系统，而是统一跨链架构下针对不同链能力的两种实现方式。它们都服务于同一个目标：使跨链交易的完成依赖同一个秘密条件，并在失败时提供安全退出路径。两种机制的选择根植于目标链的交易模型差异。

EVM 类链采用账户模型，具备较强的智能合约表达能力。合约可以维护全局状态映射（如 swapId → CrossChainSwap 结构），在链上直接保存 hashLock、timeLock、sender、recipient、amount 等字段，并通过合约函数显式执行锁定、条件领取和超时退款逻辑。整个跨链交换的状态机可以完整地在链上表达和验证，因此 HTLC 是合约链上自然且高效的选择。

BTC 采用 UTXO 模型，资产的花费由附着在 UTXO 上的锁定脚本（scriptPubKey）和花费时提供的解锁脚本（scriptSig/witness）共同控制。BTC 没有全局合约状态，每笔交易独立消费输入并产生新输出，不存在 EVM 式的持久化状态映射。虽然 BTC Script 支持 OP_SHA256 和 OP_CHECKLOCKTIMEVERIFY，能够实现基本的 HTLC，但这种方式会在链上暴露显式的条件分支脚本，且无法像智能合约那样维护跨链请求的结构化状态。Schnorr adaptor signature 将条件锁定逻辑从链上脚本转移到链下签名协议中，交易的完成取决于签名是否被正确补全，而非脚本分支的执行路径。这种方式更符合 BTC "由签名授权花费"的原生交易模型，链上只呈现一笔普通的单签名交易，既减少了脚本复杂度，又提升了交易隐私性。

因此，本文系统的兼容性并不来自”所有链使用同一种机制”，而来自”所有链遵循同一角色流程，并根据自身特性选择合适的底层机制”。在架构层，Alice、Manager、Notary、Bob 的流程保持一致；在机制层，合约链采用链上 HTLC，BTC 采用 adaptor signature 以获得更好的隐私性和链上简洁性；在协调层，swapId、hashLock、secret、amount 和 recipient 构成统一上下文。

## 第五章 系统安全性分析

### 5.1 原子性与可退款性分析

跨链资产交换最重要的安全目标是原子性，即交换要么两侧都完成，要么双方都能够回到安全状态。本文系统通过两种路径实现这一目标，分别对应合约链的 HTLC 机制和 BTC 场景的 adaptor signature 机制。

#### HTLC 路径的原子性

在 HTLC 路径中，源链和目标链都使用相同 hashLock 创建锁定，两侧领取条件被同一个 secret 绑定。正常完成时，Bob 在目标链提交 secret 领取资产，secret 随之公开上链；源链参与方观察到 secret 后，可以用同一 secret 完成源链结算。失败时，secret 未公开，任何一方都无法领取资产，timeLock 到期后双方分别退款。

原子性的关键约束在于 timeLock 的不对称设置。目标链 ManagerB 的 timeLock 必须严格短于源链 Alice 的 timeLock。原因如下：Bob 需要在目标链 timeLock 到期前提交 secret 领取资产；secret 公开后，源链参与方需要在源链 timeLock 到期前使用该 secret 完成结算。如果两侧 timeLock 相同或目标链更长，则可能出现以下攻击场景——Bob 在目标链领取资产公开 secret，但源链 timeLock 已经到期，Alice 已经退款，导致 Bob 获得目标链资产的同时 Alice 也取回了源链资产，破坏原子性。因此，合理的设置应满足：timeLock_source > timeLock_target + 安全余量，其中安全余量需覆盖链上交易确认时间和参与方的响应延迟。

#### Adaptor signature 路径的原子性

在 adaptor signature 路径中，原子性由签名补全与秘密提取的密码学绑定保证。Alice 和 Bob 分别为各自链上的交易创建预签名，预签名与同一个秘密点 T = tG 绑定。当持有秘密 t 的一方补全签名并广播交易时，另一方可以从链上出现的有效签名与预签名的差值中提取 t，进而补全自己一侧的签名完成交易。

这一路径的原子性不依赖链上合约验证 secret，而依赖以下密码学保证：不知道 t 的一方无法补全预签名（不可伪造性）；一旦补全后的签名上链，t 必然可被提取（秘密提取必然性）。失败退出则通过预先签署的 timeLock 退款交易实现——如果在约定时间内没有任何一方广播补全后的签名，双方可以广播退款交易取回资产。与 HTLC 路径类似，退款交易的 timeLock 同样需要不对称设置，确保一方完成交易后另一方有足够时间提取秘密并完成己侧交易。

#### 原子性的前提假设

两种路径的原子性都依赖以下前提：哈希函数和离散对数问题的计算安全性；链上交易能够在合理时间内被确认；参与方能够及时监听链上状态变化并在 timeLock 到期前做出响应。在这些前提下，系统能够保证跨链交换不会出现一方无条件获利而另一方无法退出的情况。

### 5.2 Notary 信任边界分析

Notary 在系统中负责监听源链事件并协调目标链响应，是跨链流程的关键中间环节。本节通过分析 Notary 可能的恶意行为及系统的对应防御，明确其信任边界。

#### 攻击模型

假设 Notary 为理性攻击者，可能采取以下恶意行为：

拒绝服务：Notary 不转发源链事件，导致目标链 ManagerB 无法创建对应锁定，跨链流程停滞。系统防御：Alice 的源链资产受 timeLock 保护，到期后可无条件退款。Notary 的拒绝服务只影响可用性（交换无法完成），不影响资产安全性（Alice 不会损失资产）。

参数篡改：Notary 向目标链转发错误的 recipient、amount 或 hashLock。系统防御：如果 hashLock 被篡改，Bob 持有的 secret 无法匹配目标链锁定，交换无法完成，最终双方退款；如果 recipient 被篡改为攻击者地址，攻击者仍然需要正确 secret 才能领取，而 secret 由 Alice 生成并控制；如果 amount 被篡改为更小值，Bob 可以拒绝接受（链下协商层面），或者系统通过源链事件与目标链锁定的参数校验发现不一致。

重放攻击：Notary 对同一笔源链事件重复向目标链发起锁定请求。系统防御：合约要求同一 hashLock 不能重复创建 ManagerLock，重复请求会被拒绝。

抢跑攻击：Notary 观察到 secret 后抢先在目标链领取资产。系统防御：目标链 ManagerLock 的 recipient 字段限定了领取方，即使 Notary 知道 secret，也无法将资产转移到非指定地址。

#### 信任边界总结

Notary 的权力被限定为"消息传递和流程触发"，不具备绕过 hashLock 领取资产、修改合约状态或阻止退款的能力。其主要风险是可用性风险（拒绝服务导致交换失败）和协调正确性风险（参数错误导致流程异常），而非直接的资产盗取风险。后续可通过多 Notary 节点、多签触发或门限签名机制降低单点可用性风险。

### 5.3 Manager 约束分析

Manager 负责资金池管理，持有或控制一定流动性，是系统中权限较高的角色。本节分析 Manager 可能的恶意行为及合约层面的约束机制。

#### 攻击模型

假设 Manager 为恶意参与方，可能采取以下行为：

拒绝响应：ManagerB 收到 Notary 的协调请求后拒绝创建目标链锁定。影响：跨链流程无法推进，但 Alice 的源链资产受 timeLock 保护，到期后可退款。Manager 的拒绝响应等价于 Notary 拒绝服务的效果，只影响可用性。

伪造锁定参数：ManagerB 创建锁定时使用错误的 recipient 或 amount。系统防御：如果 recipient 错误，真正的 Bob 无法领取，secret 不会被公开，最终双方退款；后端和前端可以通过比对源链事件与目标链锁定参数发现不一致，提醒用户不要公开 secret。

抢先领取：Manager 试图在 Bob 之前使用 secret 领取目标链资产。系统防御：completeManagerLock 函数将资产释放给 ManagerLock 中记录的 recipient 地址，而非调用者地址。即使 Manager 调用完成函数，资产仍然转给 Bob。

绕过 hashLock 直接释放：Manager 试图不经过 secret 验证直接转移锁定资产。系统防御：合约中资产释放逻辑要求 keccak256(secret) == hashLock 验证通过，且状态标记为 completed 后才执行转账。Manager 的 onlyManager 权限仅允许其创建锁定和超时退款，不能绕过完成条件。

密钥泄露：Manager 的私钥被第三方获取。影响：攻击者可以以 Manager 身份创建任意锁定或执行退款，但仍然无法绕过 hashLock 领取已锁定资产。主要风险是攻击者可以在 timeLock 到期前抢先退款，导致 Bob 无法领取。缓解措施包括多签管理、硬件密钥和限额控制。

#### 约束总结

Manager 的行为受合约状态机约束：创建锁定需要指定完整参数且 hashLock 不可重复使用，完成交易需要正确 secret，退款需要 timeLock 到期且交易未完成。Manager 不能单方面绕过这些条件获取用户资产。其剩余风险主要在运营层面（流动性不足、响应延迟、密钥管理），可通过多签、限额和监控机制进一步缓解。

### 5.4 异构链兼容性与安全等价性分析

本文系统在合约链上使用 HTLC，在 BTC 场景使用 adaptor signature，两种机制在实现层不同但服务于相同的安全目标。本节分析两种机制在安全性保证上的等价性以及多机制兼容可能引入的风险。

#### 安全等价性

HTLC 路径的安全性依赖：哈希函数的抗原像性（不知道 secret 无法通过 hashLock 验证）和 timeLock 的时间约束（到期前无法退款，到期后可无条件退款）。Adaptor signature 路径的安全性依赖：离散对数问题的困难性（不知道秘密 t 无法补全预签名）和预签署退款交易的时间约束（到期前退款交易无效，到期后可广播退款）。

两种机制在抽象层面提供相同的安全保证：完成条件绑定（一方完成交易必然导致秘密公开，使另一方具备完成条件）和失败安全退出（超时后双方可独立退款）。差异在于密码学假设不同——HTLC 依赖哈希函数安全性，adaptor signature 依赖椭圆曲线离散对数假设——但两者在当前计算能力下均被认为是安全的。

#### 跨机制秘密传递

当跨链交换的两侧分别使用不同机制时（例如源链为 HTLC，目标链为 adaptor signature），需要确保秘密能够在两种机制之间正确传递。在本文系统中，HTLC 使用的 secret 与 adaptor signature 使用的秘密 t 可以建立对应关系：令 hashLock = H(secret) 且 T = tG，其中 secret 和 t 可以是同一个值或通过确定性派生关联。当 Bob 在 adaptor signature 侧补全签名广播交易后，Alice 从链上提取 t，即可作为 secret 在 HTLC 侧完成领取。这一跨机制传递的正确性依赖于 secret/t 的一致性约定，系统在初始化阶段需确保两侧使用的秘密值相互对应。

#### 多机制兼容的潜在风险

多机制兼容引入的额外风险主要包括：时间窗口不匹配（两侧链的出块速度和确认时间不同，可能导致一侧 timeLock 到期时另一侧仍有未确认交易）；秘密格式不兼容（HTLC 的 secret 通常为 bytes32，adaptor signature 的 t 为椭圆曲线标量，需要确保编码和哈希方式一致）；监听延迟（从 BTC 链上提取签名差值需要等待交易确认，期间源链 timeLock 持续消耗）。这些风险可以通过充分的 timeLock 安全余量、统一的秘密编码规范和及时的链上监听机制加以缓解。

## 第六章 系统实现与展示

### 6.1 智能合约实现

智能合约模块是本文系统在合约链上的核心实现。`CrossChainTokenPool.sol` 同时承担资产池、跨链中间代币和 HTLC 状态机功能。合约的实现重点不是单次转账，而是维护跨链交换在不同阶段的状态：用户侧锁定由 `CrossChainSwap` 表示，目标侧响应锁定由 `ManagerLock` 表示。两个结构体的核心字段如下。

```solidity
struct CrossChainSwap {
    bytes32 hashLock;
    uint256 timeLock;
    address sender;
    address recipient;
    uint256 amount;
    bool completed;
    bool refunded;
    bool isManagerLocked;
}

struct ManagerLock {
    bytes32 hashLock;
    uint256 timeLock;
    address recipient;
    uint256 amount;
    bool completed;
    bool refunded;
}
```

其中，`hashLock` 用于约束领取条件，`timeLock` 用于约束退款条件，`completed` 和 `refunded` 用于避免同一笔交换重复完成或重复退款。用户发起跨链交换后，合约将资产转入合约地址并写入 `CrossChainSwap`；Notary 或目标侧执行节点响应后，合约创建 `ManagerLock`，使目标链资产与源链锁定通过同一个 `hashLock` 关联。

核心接口可以概括为以下几类。

```solidity
function initiateCrossChain(
    bytes32 hashLock,
    address recipient,
    uint256 amount,
    uint256 timeLockBlocks
) external returns (bytes32 swapId);

function managerRespondToCrossChain(
    bytes32 hashLock,
    address recipient,
    uint256 amount,
    uint256 timeLockBlocks
) external;

function completeCrossChain(bytes32 swapId, bytes32 secret) external;
function completeManagerLock(bytes32 hashLock, bytes32 secret) external;

function refundCrossChain(bytes32 swapId) external;
function refundManagerLock(bytes32 hashLock) external;
```

这些接口分别对应跨链流程中的发起、响应、完成和退款四个阶段。完成阶段的关键检查是 `keccak256(abi.encodePacked(secret)) == hashLock`，只有提交正确 secret 才能改变交易状态；退款阶段的关键检查是当前区块高度已经超过 `timeLock`，并且交易尚未完成。合约还通过 `CrossChainInitiated`、`CrossChainCompleted`、`ManagerLockCreated`、`ManagerLockCompleted`、`CrossChainRefunded` 和 `ManagerLockRefunded` 等事件向链下模块暴露状态变化。Notary 监听这些事件后即可驱动目标链执行对应操作。

### 6.2 后端服务实现

后端服务主要负责跨链协调。由于不同链之间不能直接通信，后端需要连接各链节点，调用相应合约或链 SDK，并维护跨链请求状态。系统中的 Python Flask 服务提供 API 接口，同时包含多链 coordinator，用于处理 ETH、BSC、Conflux、Fabric、百度超级链和 BTC 等模块之间的交互。

通用跨链接口包括：

```text
POST /cross-chain/initiate
POST /cross-chain/complete/<swap_id>
POST /cross-chain/refund/<swap_id>
GET  /cross-chain/status/<swap_id>
GET  /cross-chain/active
```

对于 BTC 适配场景，后端还提供独立入口：

```text
POST /cross-chain/eth-to-btc/initiate
POST /cross-chain/eth-to-btc/complete/<hash_lock>
GET  /cross-chain/eth-to-btc/status/<hash_lock>
```

后端的核心流程可以用伪代码表示如下。

```text
receive cross-chain request
    -> generate or receive secret/hashLock/swapId
    -> call source-chain TokenPool to lock asset
    -> wait for transaction receipt
    -> extract CrossChainInitiated event
    -> notify target-chain manager or contract
    -> record swap status
    -> expose status to frontend
```

在实现中，后端会从交易回执中解析 `CrossChainInitiated` 事件，提取 `swapId`，并根据目标链类型选择对应的处理逻辑。对于 EVM 类链，后端通过 Web3 构造并发送合约交易；对于 Fabric、百度超级链等非 EVM 链，则通过对应 SDK 或脚本封装进行调用；对于 ETH 到 BTC 的流程，则交由 BTC coordinator 处理 adaptor signature 相关状态。

需要强调的是，后端服务并不是安全性的唯一来源。它可以帮助自动化流程，提高用户体验，但资产能否释放仍由合约或签名协议决定。因此，即使后端出现异常，系统仍应依靠 timeLock 提供退款路径。这一点与第五章的 Notary 信任边界分析一致：后端与 Notary 更接近流程协调层，而不是最终资产安全的唯一保证。

### 6.3 前端可视化实现

前端模块用于展示系统流程和用户交互。项目中包含跨链交换页面和 HTLC 演示页面。跨链交换页面展示源链、目标链、跨链模式、接收地址和金额等输入项，体现产品层面的跨链入口。当前该页面主要是前端原型，并不代表真实链上交易已经执行。

HTLC 演示页面围绕 Alice - ManagerA - Notary - ManagerB - Bob 展开，用户可以通过“推进”按钮逐步观察跨链流程，包括生成 secret、计算 hashLock、源链锁定、Notary 监听、目标链响应、揭示 secret、Bob 领取和超时退款等阶段。前端状态模型如下。

```typescript
interface HtlcDemoState {
  scenario: 'complete' | 'refund';
  currentStep: number;
  completed: boolean;
  refunded: boolean;
  secret: string;
  hashLock: string;
  timeLock: string;
  swapId: string;
  txHash: string;
}
```

该状态模型与协议流程一一对应：`scenario` 区分完成路径和退款路径，`currentStep` 表示当前执行阶段，`completed` 与 `refunded` 表示最终状态，`secret`、`hashLock`、`timeLock` 和 `swapId` 则用于展示 HTLC 的关键变量。页面右侧同步展示这些状态和余额变化，使抽象的跨链协议更加直观。在论文中，前端可视化可以作为系统展示和教学辅助工具。它不替代真实链上交易，但能够帮助说明系统角色、流程顺序和安全条件，是论文中展示工程完整性的有效材料。

### 6.4 实验流程设计

为了验证系统功能和安全性，本文分别从功能正确性、安全约束和异构适配三个角度进行实验。合约链实验在本地 Hardhat 网络上执行，测试工具为 Hardhat、ethers.js 和 Chai，测试文件为 `test/HTLC.test.js`。实验中部署两个 `CrossChainTokenPool` 实例，分别模拟源链 ETH 池和目标链 BSC 池；测试账户包括 manager、Alice 和 Bob；Alice 与 manager 预先通过 `swapNativeForToken` 获得测试代币，实验金额设置为 200 至 500 个测试代币，`secret` 由 `ethers.randomBytes(32)` 生成，`hashLock` 由 `ethers.keccak256(secret)` 计算得到。BTC/scriptless 实验在 Python 模拟环境中执行，测试文件为 `schnorr/test_scenario.py`，依赖 ECPy 椭圆曲线库。

#### 功能正确性验证

功能正确性实验验证正常跨链路径是否能够完成。实验步骤为：Alice 在源链调用 `initiateCrossChain` 发起跨链请求，合约产生 `CrossChainInitiated` 事件；目标链执行 `managerRespondToCrossChain` 创建 `ManagerLock`；随后 Alice 调用 `completeCrossChain` 揭示 secret，Bob 使用同一 secret 调用 `completeManagerLock` 领取目标链资产。实际测试结果显示，交易依次触发 `CrossChainCompleted` 和 `ManagerLockCompleted` 事件，源链 `CrossChainSwap.completed` 与目标链 `ManagerLock.completed` 均变为 `true`，Bob 的目标链余额增加到测试设定的 `swapAmount`。该实验对应第五章的原子性分析，说明同一个 secret 能够联动两侧状态完成。

退款路径实验验证失败退出能力。实验中 Alice 发起跨链锁定后不公开 secret，测试脚本通过 Hardhat 时间辅助函数推进区块高度，使 timeLock 到期。随后 Alice 调用 `refundCrossChain`，manager 调用 `refundManagerLock`。实际结果显示，合约分别触发 `CrossChainRefunded` 和 `ManagerLockRefunded` 事件，Alice 与 manager 的测试代币余额恢复到初始水平。该结果对应第五章 5.1 中的可退款性说明，表明当 secret 未公开时，系统不会造成资金永久锁定。

#### 安全约束验证

安全约束实验主要验证 hashLock、权限控制和重复锁定防护。重复 hashLock 实验中，manager 第一次使用某一 `hashLock` 创建 `ManagerLock` 后，再次使用同一 `hashLock` 创建锁定会被合约拒绝，实际 revert 信息为 `Hash lock already used`。该结果说明合约能够限制同一哈希锁被重复响应，降低重放和重复领取风险。

权限控制实验中，非授权账户尝试调用 `managerRespondToCrossChain`，交易被合约回滚。当前合约实现中该函数使用 `onlyNotary` 修饰符，因此实际回滚信息为 `Only notary can call this function`。这表明目标链响应动作受到角色权限限制，与第五章对 Notary 信任边界的分析相对应。测试脚本中原有断言使用的是旧错误信息 `Only manager can call this function`，因此自动化测试输出表现为 5 个用例通过、1 个用例因错误信息断言不一致而失败；从合约行为看，非授权调用被拒绝这一安全约束已经生效，后续应同步更新测试脚本中的权限命名。

错误 secret 的安全性可由完成函数的条件检查推出：`completeCrossChain` 与 `completeManagerLock` 均要求提交的 secret 经哈希后等于锁定时保存的 `hashLock`。若提交错误 secret，交易无法通过该 require 条件并回滚。该实验与第五章 5.1 的 hashLock 原子性约束相对应，用于说明不知道 secret 的参与方无法完成领取。

#### 异构适配验证

BTC/scriptless 实验验证 adaptor signature 的机制流程。实验脚本依次生成 Alice、Bob、Carol 的 secp256k1 公私钥，构造聚合锁定点，生成 Alice 到 Bob 以及 Bob 到 Carol 的 adaptor signature。随后 Carol 补全签名并广播模拟交易，Bob 从该交易中提取秘密，再使用该秘密完成 Alice 到 Bob 的结算。实际运行输出显示：

```text
Tx_Bob_to_Carol confirmed on-chain.
Carol 成功领取资金。
Bob 从交易 Tx_Bob_to_Carol 中提取到了秘密。
Tx_Alice_to_Bob confirmed on-chain.
Bob 成功领取资金。
Simulation Complete.
```

该实验说明在 BTC/scriptless 场景中，交易完成与秘密提取之间能够建立联系。它对应第五章 5.4 的异构兼容分析：合约链通过 HTLC 在链上验证 secret，BTC/scriptless 场景通过签名补全关系暴露秘密，两者虽然实现方式不同，但都服务于“完成条件绑定”和“失败可退出”的安全目标。

综上，第六章实验与第五章安全性分析形成对应关系：正常完成与退款实验验证 5.1 的原子性和可退款性；权限控制实验验证 5.2 的 Notary 信任边界；重复 hashLock 和错误 secret 实验验证 5.3 中对 Manager 行为和重放风险的约束；adaptor signature 模拟实验验证 5.4 中多机制兼容的可行性。

## 第七章 总结与展望

### 7.1 工作总结

本文围绕跨链资产交换问题，设计并实现了一个基于公证人协调与哈希时间锁的原型系统。系统采用 Alice - ManagerA - Notary - ManagerB - Bob 的角色模型，将用户发起、资金池锁定、事件监听、目标链响应和资产领取等阶段有序组织起来。在支持智能合约的链上，系统通过 HTLC 实现 hashLock 条件领取和 timeLock 超时退款；在 BTC 场景中，系统通过 Schnorr adaptor signature 将条件锁定放入签名协议，相比链上 HTLC 脚本提供了更好的隐私性和链上简洁性。

通过上述设计与实现，本文形成了一套较为完整的跨链资产交换原型。系统层面，论文将跨链流程抽象为 Alice、ManagerA、Notary、ManagerB 和 Bob 之间的协作关系，使用户请求、资金池锁定、公证中继和目标链释放之间的边界更加清晰。机制层面，系统在合约链上实现了 HTLC 状态机，能够支持资产锁定、secret 验证、交易完成和超时退款；同时将 Schnorr adaptor signature 纳入统一架构，用于说明 BTC 场景下通过签名补全实现条件锁定的可行性。实现层面，项目完成了智能合约、后端协调服务、BTC 模拟模块和前端可视化页面的组合，使协议流程不仅能够从理论上说明，也能够通过代码结构和实验过程进行展示。

### 7.2 系统不足

本文系统仍然属于原型实现，存在一定不足。首先，BTC 模块以机制模拟为主，尚未完整接入真实 BTC 测试网或主网交易流程。其次，Notary 当前更接近单节点协调模型，虽然合约和 timeLock 可以限制其直接资产风险，但其可用性仍可能影响跨链体验。再次，Manager 资金池的流动性管理较为简单，尚未充分考虑复杂汇率、手续费、滑点、限额和风险控制。最后，系统安全性主要通过逻辑分析和实验验证说明，尚未进行形式化证明和完整安全审计。

### 7.3 后续展望

后续工作可以围绕安全性、真实性和可扩展性继续推进。在安全性方面，当前 Notary 更接近单节点协调模式，后续可以引入多 Notary 节点、多签触发或门限签名机制，使跨链消息转发不再依赖单一节点的可用性。在真实性方面，BTC 模块仍以模拟验证为主，后续需要接入 BTC 测试网交易流程，将 adaptor signature 从本地脚本验证扩展到真实 UTXO 花费和链上确认过程。在系统能力方面，Manager 资金池仍可以进一步完善，例如加入流动性监控、动态费率、额度限制和异常告警机制，以提高系统面对不同交易规模和市场波动时的稳定性。随着更多异构链的接入，系统还需要进一步抽象链适配接口，使 EVM 链、联盟链和 scriptless 场景能够以更统一的方式接入跨链协调层。除此之外，HTLC 与 adaptor signature 的安全性还可以通过形式化建模进一步论证，从而更严格地分析系统在延迟、抢跑、节点失效和参数错配等攻击场景下的安全边界。

总体而言，本文系统展示了一种将公证人协调、HTLC 哈希时间锁和 Schnorr adaptor signature 结合起来的跨链资产交换方案。该方案在统一角色架构下兼容不同链的执行能力和隐私需求，能够为异构区块链之间的资产互操作提供有价值的原型参考。


流程图（第三章）
实现与展示（第六章）--代价：时间/存储（性能分析，notary活跃时长）
首个中继节点无需资金支持（introduction, 分析）
无脚本区块链有哪些：门罗币，ZCash（纯密码学）
无脚本区块链好处（存在性） -- 劣势：无法迁移 -- adaptor sig
使用hash不完备的BTC充当scriptless区块链（不影响正确性）（不用说BTC不适用于HASH）