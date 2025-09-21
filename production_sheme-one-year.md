# RWA 平台技术方案（Solana Year-1 → Sui Year-2）

> 目标：1 年内在 **Solana** 跑通发行—认购—受限转让—收益分配—赎回全闭环；第 2 年追加 **Sui（Move）** 平行部署与跨链镜像。本文档可直接放入 GitHub 仓库作为项目蓝本（PRD + 技术方案 + 里程碑）。

---

## 目录

- [1. 背景与边界](#1-背景与边界)  
- [2. 总体架构](#2-总体架构)  
- [3. Solana 链上 Program 设计](#3-solana-链上-program-设计)  
  - [3.1 Program 列表与职责](#31-program-列表与职责)  
  - [3.2 账户与 PDA 设计](#32-账户与-pda-设计)  
  - [3.3 受限转让：Token-2022 Transfer Hook](#33-受限转让token2022-transfer-hook)  
  - [3.4 回退方案（经典 SPL）](#34-回退方案经典-spl)  
  - [3.5 指令流示例（Anchor 伪 IDL）](#35-指令流示例anchor-伪-idl)  
- [4. 链下系统与 App](#4-链下系统与-app)  
- [5. 数据模型（链下）](#5-数据模型链下)  
- [6. MPC 与金库](#6-mpc-与金库)  
- [7. AI Agent 场景](#7-ai-agent-场景)  
- [8. 里程碑与排期（12 个月）](#8-里程碑与排期12-个月)  
- [9. 工作量、团队与审计](#9-工作量团队与审计)  
- [10. Year-2：Sui（Move）并行方案](#10-year2sui-move并行方案)  
- [11. 跨链与一致性策略](#11-跨链与一致性策略)  
- [12. 落地 Checklist](#12-落地-checklist)  
- [13. PoC（前 2–4 周）任务清单](#13-poc前-24-周任务清单)  
- [14. 仓库结构建议](#14-仓库结构建议)  
- [15. 配置与参数表（示例）](#15-配置与参数表示例)  
- [16. 风险与控制点](#16-风险与控制点)  
- [17. 许可证与合规声明](#17-许可证与合规声明)

---

## 1. 背景与边界

- **资产类型**：以新能源/光伏收益权为主（可扩展其他 RWA）。  
- **目标**：  
  - Year-1（Solana）：发行、认购、受限转让、收益分配、赎回、信息披露、审计追踪。  
  - Year-2（Sui）：等价模块平行部署 + 跨链镜像或消息证明。  
- **合规**：KYC/AML、地域限制、锁定期、黑名单与额度控制（技术实现角度描述，非法律意见）。  
- **钱包/密钥**：平台金库 MPC + 多签；用户自管或轻托管二选一。

---

## 2. 总体架构

- **链上（Solana Year-1）**：  
  - Program：Access/Config、KYC Registry、Compliance（Transfer Hook）、Asset Mint Manager、Offering/Subscription、Treasury/Fee、Distribution（Merkle Claim）、Buyback/Redemption、Oracle Adapter、Document Registry、Pause/Emergency、（可选）OTC/撮合。  
  - 预言机：Pyth / Switchboard。  
- **链下**：KYC/AML、发行人/投资者/合规/运营门户、账务与对账、税费、IR 看板、监控告警。  
- **App**：Flutter（iOS/Android）+ 原生签名/推送插件。  
- **AI**：尽调/合规清单/投资者助手/运维预警 Agent。  

---

## 3. Solana 链上 Program 设计

### 3.1 Program 列表与职责

| # | Program | 作用 | 必选 |
|---|---|---|---|
| 1 | **Access & Config** | 全局参数与角色、紧急暂停、费率 | ✅ |
| 2 | **KYC & Identity Registry** | 地址→KYC/地域/合格/锁定期/黑名单 | ✅ |
| 3 | **Compliance Transfer Hook** | Token-2022 转账前检查（受限转让） | ✅（优先） |
| 4 | **Asset Mint Manager** | 创建/管理资产 Mint，Metadata 绑定 | ✅ |
| 5 | **Offering / Subscription** | 认购池、价格、额度、打款与配售 | ✅ |
| 6 | **Treasury / Fee Vault** | 募集与费用金库、白名单提现 | ✅ |
| 7 | **Distribution (Merkle)** | 分配批次存证、投资者 `claim` | ✅ |
| 8 | **Buyback / Redemption** | 回购/赎回、到期处理 | ✅ |
| 9 | **Oracle Adapter** | Pyth/Switchboard 签名校验与落库 | ✅ |
|10 | **Document Registry** | 文档哈希/URI/签名上链 | ⭕️ |
|11 | **Pause / Emergency** | 平台/资产级暂停 | ✅（可并入 #1） |
|12 | **OTC/撮合** | 合规托管式撮合与结算 | ⭕️（法规允许时） |

> 规模：核心 **7–9 个** Program 即可跑闭环；完整形态 **8–12 个**。

### 3.2 账户与 PDA 设计

- `GlobalConfigPDA`：平台参数、管理员/合规/金库/喂价角色、公钥白名单、费率、暂停状态。  
- `KycRecordPDA(user)`：KYC 状态、地域代码、合格标记、锁定期、额度；事件 `KycApproved/Revoked`。  
- `AssetMintPDA(asset_id)`：SPL Mint、Metadata、发行参数、默认锁定期模板。  
- `OfferingPDA(asset_id, round_id)`：价格、最小/最大配售、截止时间、资金金库。  
- `TreasuryVaultPDA(asset_id)`：USDC/资产金库（多签/MPC 白名单）。  
- `DistributionPDA(batch_id)`：Merkle Root、总额、快照高度/时间、税费口径。  
- `ClaimStatusPDA(batch_id, user)`：已领取标记与数量。  
- `OraclePDA(source_id)`：数据源、最后喂数信息、阈值。  

### 3.3 受限转让：Token-2022 Transfer Hook

- 在 `transfer` 前回调 **Compliance Program**，校验：  
  1) `from/to` 均已通过 KYC 且未在黑名单；  
  2) 地域与锁定期合规；  
  3) 金额/频率不超限；  
  4) 平台或资产未暂停。  
- 参数化“豁免白名单”（如托管账户/做市账户）。  
- **灰度开关**：可切换检查强度，便于灰度发布与应急。

### 3.4 回退方案（经典 SPL）

- 若钱包/交易所对 Token-2022 支持不足：  
  - 使用 **冻结权限 + 托管撮合**：转让经合规模块托管账户结算；  
  - 对合规账户启用**冻结豁免**；  
  - 保持与 Hook 版同一套合规规则，便于未来切换。

### 3.5 指令流示例（Anchor 伪 IDL）

<details>
<summary>创建资产与发行（节选）</summary>

```ts
// 1) Admin 初始化全局配置
instruction InitializeConfig {
  accounts: [GlobalConfigPDA, Admin, SystemProgram]
  args: { fee_bps: u16, pause: bool, roles: Pubkey[] }
}

// 2) 合规侧登记用户 KYC
instruction UpsertKycRecord {
  accounts: [GlobalConfigPDA, KycRecordPDA(user), ComplianceSigner]
  args: { user: Pubkey, region: u16, accredited: bool, lock_until: i64, limit: u64, blacklist: bool }
}

// 3) 创建资产与发行参数
instruction CreateAssetMint {
  accounts: [GlobalConfigPDA, AssetMintPDA(asset_id), Mint, Metadata, Issuer, SystemProgram, TokenProgram]
  args: { name: string, symbol: string, decimals: u8, total_supply: u64, default_lock_days: u16 }
}

// 4) 开启认购轮
instruction OpenOffering {
  accounts: [OfferingPDA(asset_id, round_id), TreasuryVaultPDA(asset_id), Issuer]
  args: { price_usdc: u64, min_allocation: u64, max_allocation: u64, end_time: i64 }
}

// 5) 投资者申购
instruction Subscribe {
  accounts: [OfferingPDA, TreasuryVaultPDA, Investor, InvestorUsdcATA, SystemProgram, TokenProgram]
  args: { amount_usdc: u64 }
}

// 6) 结算配售并铸份额
instruction SettleAndMint {
  accounts: [OfferingPDA, AssetMintPDA, InvestorTokenATA, Issuer, MintAuthorityPDA, TokenProgram]
  args: { allocation: u64 }
}

// 7) 发布分配批次（Merkle）
instruction PublishDistribution {
  accounts: [DistributionPDA(batch_id), Issuer, SystemProgram]
  args: { merkle_root: [u8;32], total: u64, snapshot_slot: u64, tax_schema: u8 }
}

// 8) 投资者领取
instruction ClaimDistribution {
  accounts: [DistributionPDA, ClaimStatusPDA(user), TreasuryVaultPDA, Investor, InvestorUsdcATA, TokenProgram]
  args: { amount: u64, proof: Vec<[u8;32]> }
}
```
</details>

---

## 4. 链下系统与 App

- **投资者门户**：注册、KYC、资产目录、认购、持仓、分配领取、提现、税表导出。  
- **发行人门户**：项目上架、材料上传、公告、开启/结算认购、发布分配、赎回。  
- **合规后台**：KYC 审核、地域/合格标记、黑名单、锁定期、风险报表。  
- **运营后台**：参数/费率、预言机数据管理、对账与财务、审计导出。  
- **App（Flutter）**：账户、KYC、签名、推送、持仓、领取、提现；原生插件对接系统权限与密钥。  

---

## 5. 数据模型（链下）

```text
users(id, email, wallet_pubkey, kyc_status, region, accredited, created_at)
kyc_records(user_id, status, reviewer, evidence_uri, updated_at)
investor_eligibility(user_id, net_worth, income, expiry_at)

assets(id, code, name, decimals, total_supply, issuer_id, default_lock_days, metadata_uri)
offerings(id, asset_id, round_id, price_usdc, min_alloc, max_alloc, end_time, status)
subscriptions(id, offering_id, user_id, amount_usdc, allocation, status, tx_sig)

positions(id, user_id, asset_id, balance, last_snapshot_slot)
distributions(id, asset_id, batch_id, merkle_root, total, snapshot_slot, tax_schema)
claims(id, batch_id, user_id, amount, claimed_at, tx_sig)

treasury_ledger(id, asset_id, flow_type, token, amount, tx_sig, created_at)
reconciliation(id, date, source, ok, diff, report_uri)

oracles(id, source, symbol, latest_value, last_signed_at, signer_pubkey)
production_metrics(id, asset_id, ts, energy_kwh, price, revenue, proof_uri)

documents(id, asset_id, kind, sha256, uri, signer, onchain_ref)
audit_trail(id, actor, action, ref, metadata_json, ts)
```

---

## 6. MPC 与金库

- 平台金库：**MPC（TSS）+ 多签（如 Squads）+ 白名单地址**，提现限额与审批流。  
- 用户侧：自管钱包或轻托管（邮箱/手机恢复 + 社交恢复）；提现白名单与限额。  
- 监控：余额阈值告警、失败交易重试、地址画像与风险评分。

---

## 7. AI Agent 场景

- **尽调 Agent**：合同要点抽取、风险条款清单、材料缺失检测。  
- **合规 Agent**：大额转让检查清单、KYC 二审辅助。  
- **投资者助手**：基于链上事件与文档的 QA、收益解释。  
- **运维 Agent**：产能异常与保修到期预警，自动生成工单。

---

## 8. 里程碑与排期（12 个月）

| 阶段 | 时间 | 目标/交付 |
|---|---|---|
| **Phase 0** | 2–4 周 | 选型（Solana + Token-2022 优先、SPL 回退），PoC：KYC+合规模块+资产发行+简易分配；威胁建模与审计排期 |
| **Phase 1** | 月 1–3 | MVP 闭环：Access/Config、KYC、Asset Mint、Offering、Treasury、Distribution v1；门户（注册/KYC/认购/持仓/提现）；小额真实发行（沙箱） |
| **Phase 2** | 月 4–6 | 合规转让（Transfer Hook/托管撮合）、Buyback/Redemption、Document Registry、IR 看板；**外部安全审计 #1** |
| **Phase 3** | 月 7–9 | 多资产/多批次、分配批量化（Merkle 工具链）、税费/报表；Flutter App 上线；AI Agent v1 |
| **Phase 4** | 月 10–12 | 稳态运营、支付渠道冗余、BCP/DR；**外部安全审计 #2**；为 Sui 做数据/接口对齐准备 |

---

## 9. 工作量、团队与审计

- **Program 数量**：核心闭环 7–9，完整 8–12；代码量 5k–10k SLOC（不含测试）。  
- **团队配置（建议 9–12 人）**：  
  - 链上 2–3（Solana/Anchor + 后续 Move）  
  - 后端 2（Go/Node/Rust 任一）  
  - 前端/Flutter 2  
  - 合规/数据 1–2  
  - DevOps/Sec 1  
  - 产品/设计/PM 1–2  
- **审计**：至少 2 次独立外部审计（发行前、赎回/二级流转前），上线后持续监控。

---

## 10. Year-2：Sui（Move）并行方案

### 10.1 包/模块建议（8–10 个）

- `access::roles`（角色与参数）  
- `identity::kyc_registry`（KYC/地域/锁定期）  
- `compliance::transfer_gate`（受限转让前检查）  
- `asset::mint_manager`（资产资源与供应、冻结/赎回状态机）  
- `offering::subscription`（申购/配售/退款）  
- `treasury::vault`（资金/费用，能力控制）  
- `distribution::merkle_claim`（批次资源、已领标记、校验）  
- `buyback::redemption`（赎回/回购）  
- `oracle::adapter`（签名校验写参）  
- `doc::registry`（文档哈希/URI）

> Move 的 `key/store/drop/copy` 能把“受限转让/已领状态/锁定期”建模为强资源，不可错配。

### 10.2 上线策略

- **镜像发行**：Sui 独立新批次，同一合规后台；两链各自清算。  
- **消息证明/映射**：用跨链消息（如 Wormhole Generic Message 或自建轻桥）把 **Solana 快照/事件**送到 Sui，铸镜像凭证（可回收）。  
- **只读联邦**：Sui 只做展示与派发，资金仍在 Solana。

---

## 11. 跨链与一致性策略

- 统一 **资产 ID/批次 ID/文档哈希** 命名；  
- KYC/AML 后台作为 **统一事实源**，写两链 registry；  
- 对齐税费/会计口径；  
- 选择 **Merkle 根/快照** 作为跨链最小可信载荷。

---

## 12. 落地 Checklist

**链上（Solana）**  
- [ ] Access/Config、Pause/Emergency  
- [ ] KYC & Identity Registry  
- [ ] Compliance Transfer Hook（或 SPL 回退托管撮合）  
- [ ] Asset Mint Manager + Metadata  
- [ ] Offering/Subscription、Treasury/Fee  
- [ ] Distribution（Merkle 批次 + Claim）  
- [ ] Buyback/Redemption  
- [ ] Oracle Adapter（Pyth/Switchboard）  
- [ ] Document Registry  
- [ ] 审计两轮 + 监控告警 + Runbooks

**链下/产品**  
- [ ] 投资者/发行人/合规/运营门户 v1  
- [ ] 对账（USDC/法币）、税费预扣、财务报表  
- [ ] IR 看板（产能/收益曲线）  
- [ ] Flutter App + 原生签名/推送插件  
- [ ] MPC 金库（阈值、白名单、审批流）  
- [ ] BCP/DR、年审资料导出

---

## 13. PoC（前 2–4 周）任务清单

- [ ] Anchor 项目骨架 + Workspace  
- [ ] Program：Config、KYC、Asset Mint（最小实现）  
- [ ] Token-2022 Mint + Transfer Hook Demo（或 SPL 回退）  
- [ ] Offering 最小闭环：USDC 入金 → 配售 → 铸份额  
- [ ] Distribution 最小闭环：生成 Merkle → 上链 root → `claim`  
- [ ] 简易门户：注册、KYC 测试页、认购/持仓展示  
- [ ] Pyth/Switchboard 接入 Demo（签名校验到上链）  
- [ ] CI：lint、单测、e2e（localnet/devnet），自动部署脚本

---

## 14. 仓库结构建议

```text
rwa/
 ├─ programs/                   # Solana Programs (Anchor)
 │   ├─ access/
 │   ├─ kyc/
 │   ├─ compliance_hook/
 │   ├─ asset_mint/
 │   ├─ offering/
 │   ├─ treasury/
 │   ├─ distribution/
 │   ├─ buyback/
 │   ├─ oracle_adapter/
 │   └─ doc_registry/
 ├─ app/                        # Web 前端（React/Vue 任一）
 ├─ mobile/                     # Flutter App
 ├─ backend/                    # API/Indexer/对账服务（Go/Node/Rust）
 ├─ agents/                     # AI Agent 工作流
 ├─ ops/                        # IaC、监控、告警、Runbooks
 ├─ specs/                      # PRD、接口、合约 IDL、Merke 文件格式
 ├─ audits/                     # 自查与外部审计报告
 └─ README.md
```

---

## 15. 配置与参数表（示例）

| 参数 | 说明 | 缺省 |
|---|---|---|
| `fee_bps` | 平台费（基点） | 50 (=0.5%) |
| `default_lock_days` | 默认锁定期（天） | 90 |
| `kyc_regions_allow` | 允许地域码列表 | 按法务配置 |
| `max_tx_amount` | 单笔转让限额 | 动态 |
| `oracle_max_delay` | 预言机数据最大延迟 | 5 分钟 |
| `pause_flags` | 平台/资产级暂停位 | 0 |
| `claim_deadline_days` | 分配领取截止 | 365 |

---

## 16. 风险与控制点

- **合规模糊**：技术实现不等于法律合规；法务先行，技术按清单落地。  
- **Token-2022 兼容性**：提供 SPL 回退与灰度开关。  
- **账户并发**：拆分指令、减少热账户、批处理/队列化。  
- **Merkle 分配可信度**：生成—审计—签名—上链 root 全链路留痕。  
- **MPC/多签运营风险**：审批流、限额、白名单、操作审计；演练 BCP/DR。  
- **预言机篡改/延迟**：多源比对、时间窗限制、阈值告警与熔断。  

---


### 附录 A：Sui（Move）模块骨架（示意）

```move
module access::roles { /* roles, params as resources */ }
module identity::kyc_registry { /* user -> KYC/region/lock/state */ }
module compliance::transfer_gate { /* before_transfer checks */ }
module asset::mint_manager { /* asset resource + supply + states */ }
module offering::subscription { /* subscribe/allocate/refund */ }
module treasury::vault { /* funds with capabilities */ }
module distribution::merkle_claim { /* batch, proofs, claimed flags */ }
module buyback::redemption { /* redeem/buyback state machine */ }
module oracle::adapter { /* sig verify + write params */ }
module doc::registry { /* doc hash/uri */ }
```

