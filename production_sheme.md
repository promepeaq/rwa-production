一、目标与边界（Year-1）

目标：支持一类主资产（光伏电站/电站份额/收益权），完成发行→认购→二级流转→收益分配→赎回/处置全闭环，合规受限转让，KYC/AML，法币/稳定币入金，月度收益结算，仪表盘与审计追踪可用。

链选型：Year-1 以 EVM 主链（如 Base/Arbitrum/Polygon 任选其一）为核心；Q3 可上 Sui 平行部署（Move 合约更强表达资产状态机），Q4 做跨链桥接/镜像。

钱包与密钥：平台 MPC 账户 + 用户自管钱包并存；机构侧采用 MPC + 硬件密钥白名单。

合规模式：支持“合格/白名单地址 + 地域限制 + 锁定期”风控规则（示例：Reg D/Reg S 风格的受限转让，仅作技术实现提示，非法律意见）。

二、系统总览与模块
2.1 业务流程（高层）

资产上链（尽调→评估→法律文件→托管→上链登记）

发行（白名单合格投资者认购，USDC/法币入金）

二级流转（受限可转，让渡/回购/场内 OTC）

收益分配（按持仓快照分配，税费预扣）

信息披露（产能、电价、现金流、审计报告、链上凭证）

赎回/处置（到期、违约事件触发工作流）

2.2 技术架构（概念）

链上层：资产发行管理、受限转让、收益分配、赎回、清算事件、价格/产能预言机、治理/参数、费用与金库。

链下服务：KYC/AML、合格投资者认证、发行人/投资者门户、文档与审计存证、账务与对账、支付网关、风控规则引擎。

钱包与托管：MPC 服务（TSS/阈值签名），托管/多签金库，提现白名单。

数据与风控：指标快照、应收/应付、收益测算、预警（产能低于阈值、违约条款）。

AI Agent：尽调文档抽取、合规检查清单、投资者问答、运维报警助理。

三、需要开发的智能合约（EVM Year-1）

建议使用 UUPS/Transparent Proxy 可升级模式 + OpenZeppelin 基础库。下列按照“合约组件（建议合约数量/功能）”列出。

AccessManager / Roles（1 个）

角色：ADMIN、ISSUER（发行人）、COMPLIANCE（合规）、ORACLE、TREASURY、PAUSER。

管理白名单服务地址、参数 Owner、升级权限。

Identity/KYC Registry（1 个）

维护投资者地址→KYC 状态、地域代码、合格投资者标签、锁定期标记。

事件：KYCApproved/KYCRevoked。链下服务负责写入。

Compliance Transfer Gate（1 个库/或 1 合约）

标准接口 canTransfer(from,to,assetId,amount)；校验白名单、地域、锁定期、黑名单、额度。

与各资产合约集成（hook）。

Asset Factory（1 个）

创建标准化 RWA-ERC20（可细分份额）或 RWA-ERC721/1155（份额凭证/NFT 证书）实例。

记录发行参数（名称、代码、总额、锁定/赎回规则）。

RWA ERC20（1–2 个实现）

受限转让（与 Compliance Gate 集成）、快照（ERC20Snapshot）、回购/赎回接口。

事件：Issue/Mint/Redeem/ForceRedeem。

RWA ERC1155（可选 1 个）

用于批次化份额或不同层级权益（如 A/B/C 档收益）。

Offering/Subscription（1 个）

认购合约：申购白名单、价格、最小/最大配售、打款资产（USDC/ETH）。

申购成功后铸造份额到投资者地址；支持超额退回。

Treasury / Fee Vault（1 个）

存放募集资金/费用；参数化费率（平台费、发行费、赎回费）与分账。

Distribution（1 个）

收益分配：按某日/某块高度快照进行 distribute(uint256 amount)；

支持可提取模式（投资者自行 claim()）和直接空投两种。

Oracle Adapter（1 个）

产能/电价/指数、汇率等链下数据的喂价/喂数入口（签名校验 + 最后写入权限 ORACLE 角色）。

Buyback/Redemption（1 个）

赎回/回购规则：按净值/估值进行兑换；到期自动化/管理员触发。

Governance / Parameters（1 个）

平台级参数与资产级参数：费率、锁定期默认值、合规策略开关；

简化治理（多签+Timelock），Year-1 不建议复杂投票制。

Pause/Emergency（1 个）

平台级紧急暂停、资产级暂停（转账、赎回、认购）。

Document Registry（1 个，选做）

重要文件（审计、保险、PPA 合同）哈希与 URI 存证。

小计（EVM Year-1）：
基础13–15 个合约/代理（含实现/工厂/库），核心逻辑代码行数通常 4k–8k SLOC（不含测试）。
审计建议：至少 2 次独立安全审计（发行前 + 首次分配/赎回前），外加持续的形式化规则测试（Foundry/Slither）。

Sui（Move）平行实现（Q3 以后）

对应模块：Identity、Compliance、Asset、Offering、Distribution、Buyback、Oracle。

Move 中建议以资源类型表达“份额”和“合规持有者”，使用**能力（key、store、drop、copy）**严格建模，8–10 个包足够 Year-1 需求。

四、链下系统与服务开发
4.1 门户与后台

投资者门户：注册/KYC、资产目录、申购、持仓、收益、提现、税表导出、链上证明。

发行人门户：项目上架、材料上传、参数配置、配售与公告、收益分配触发。

合规后台：KYC 审核、地域/资格标记、黑名单、锁定期设置、异常交易复核。

运营后台：资产参数（费率/期限）、公告、价格/产能数据管理、对账报表。

4.2 支付与清结算

入金：USDC/USDT/法币通道（第三方支付/银企直连），自动对账入库。

出金/分配：链上 claim + 司法辖区税费预扣支持（链下计算，链上记录净额）。

账务子系统：应收/应付、分润、发票、审计分录、月结对账。

4.3 KYC/AML 与合规模块

接入 1–2 家供应商（身份核验、制裁名单、PEP、地址风险评分）；

合格投资者证明（收入/净资产/机构资质）上传与人工复核；

规则引擎（地域/额度/频率/风险评分）→ 写入链上 Identity/KYC registry。

4.4 数据与预言机

产能（发电量）、电价、保单状态、维护事件、现金流入/出 → 链下采集与签名喂数；

公示看板：IR 页面展示 KPI（PR、EPC、LCOE、发电/收益曲线）。

4.5 钱包与 MPC

平台金库：MPC + 多签 + 白名单地址，提现限额与审批流；

用户：可用自管钱包；新手可选 MPC 轻托管（邮箱/手机恢复 + 社交恢复）。

风控：地址画像/灰度评分，异常交易二次验证（MPC 联合签名阈值提升）。

4.6 AI Agent（可选但强推）

尽调 Agent：自动抽取合同要点、风险条款清单、材料缺失提示。

合规 Agent：对新用户/大额转让生成检查清单，辅助合规人员二审。

投资者助手：基于文档与链上事件的 QA、收益预测解释。

运维 Agent：产能异常、保修到期预警，生成工单。

五、数据模型（核心表示例）

users、kyc_records、investor_eligibility

assets、offerings、subscriptions、positions（链上镜像+快照高度）

distributions、claims、withholdings（税费）

treasury_ledger（入金/出金/费用/分润）、reconciliation（对账）

oracles、production_metrics（产能/电价/收益）

documents（哈希/URI/签名）、audit_trail（不可篡改事件日志）

六、合规模型（技术实现角度）

受限转让：所有 transfer 前走 ComplianceGate；白名单+地域+锁定期+额度。

KYC/AML：链下完成→链上登记；变更触发事件；合规后台可撤销资格。

信息披露：文档哈希上链；重要事件（收益分配/违约）写链上事件。

风控：大额/频繁转让冻结检查；黑名单即时阻断。

注：以上为技术与产品实现说明，具体法律合规须由律师把关。

七、开发阶段与里程碑（12 个月）
Phase 0（2–4 周）— 立项与 PoC

确定链（EVM L2）与费用模型、账户体系（MPC 方案）。

PoC：RWA-ERC20 + ComplianceGate + KYC Registry + 简易 Distribution。

输出：技术架构图、威胁建模、审计厂商排期。

Phase 1（第 1–3 月）— 核心闭环 MVP

合约：1–10（见上清单前半段）+ 测试 & 内审。

门户：投资者注册/KYC、资产目录、申购、持仓页。

金库：MPC 落地，USDC 入金对账。

首次**小规模真实资产发行（沙箱）**与模拟收益分配。

Phase 2（第 4–6 月）— 二级市场与收益

合约：Distribution、Buyback/Redemption、Fee Vault 完备。

二级：受限 OTC/订单簿（链下撮合 + 链上结算）或 AMM（若法规允许）。

合规后台 v1、审计存证、IR 看板。

第一次外部安全审计 + 修复。

Phase 3（第 7–9 月）— 运维与多资产

多项目/多批次发行；Oracle 喂数上线（产能/电价）。

税费与报表、会计科目、自动月结。

iOS/Android App（Flutter）+ 原生插件（提现、推送、安全模块）。

AI Agent v1（尽调/投资者助手）。

Phase 4（第 10–12 月）— 扩展与合规深化

Sui 平行合约（可选），跨链镜像/网关。

第二次外部安全审计（赎回与二级流转重点）。

合作托管与支付渠道冗余；业务连续性演练（BCP/DR）。

年度合规审计材料导出一键化。

八、团队与预算（参考）

链上工程：2–3（Solidity/Move）

后端：2（Go/Node/Rust 任一 + 数据/对账）

前端/Flutter：2（Web + App）

合规/风控/数据：1–2（可兼职外包）

DevOps/SRE & 安全：1

产品/设计/项目管理：1–2

核心 9–12 人；外部审计 2 次、KYC/AML/支付/托管服务费用需单列。

九、测试与安全

单测/属性测试（Foundry/Hardhat + invariant）

静态/形式化工具（Slither、Mythril、Echidna）

审计前自查清单：授权最小化、资金流向、紧急刹车、升级 Timelock。

运行期监控：链上事件告警、金库余额阈值、Oracle 异常、MPC 审批流失败。

十、第一年上线还“剩下的工作清单”（落地 Checklist）

合约侧

 Access/Roles、KYC Registry、ComplianceGate

 Asset Factory + RWA Token（ERC20/1155 至少一种）

 Offering/Subscription、Treasury/Fee、Distribution、Buyback

 Oracle Adapter、Pause、Governance/Params、Document Registry

 脚本：批量白名单、配售、分配、赎回、暂停/恢复

 审计两轮 + 修复

链下/产品侧

 投资者门户 v1（注册、KYC、申购、持仓、提领）

 发行人门户 v1（上架、公告、分配触发）

 合规后台 v1（KYC 审核、锁定期/黑名单、风控报表）

 对账与账务 v1（USDC/法币、分润、税费）

 预言机链路（数据签名→上链）

 Flutter App（登录、KYC、持仓、领取、通知）

 MPC 金库落地（阈值、白名单、审批流）

 IR 看板与文档存证

 运维监控与告警

 BCP/DR 演练与合规年审包

合同/合约数量与规模小结

EVM 合约：约 13–15 个（含 Proxy），4k–8k 行核心逻辑。

Sui 包（可选）：约 8–10 个。

链下服务：4 个门户（投资者/发行人/合规/运营）+ 财务/对账/预言机/钱包网关。

AI Agent：2–4 个可用工作流（尽调、合规清单、投资者问答、运维预警）。

结语与下一步

如果你打算先做新能源 RWA，我可以把上述“通用方案”快速细化为光伏资产专用 PRD：把 PPA 合同字段、产能/补贴口径、保函/保险要素、收益模型具体化，并附上 合约接口（ABI/Move 模块签名）与链下接口（REST/GraphQL）。
你只要回复“要光伏专版”，我就直接把字段模型、事件、表结构、接口协议整套给你。
