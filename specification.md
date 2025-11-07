## 1. Dashboard页面

- **Total Value Locked**：全体用户代币持有量+总资产代币发行量*资产代币的发行价，单位为  百万美元
- **Active Power Stations**：平台上架的（Project中可直购的）资产项目总量
- **Energy Generated**：所有项目当日发电量，拿各自项目的API综合计算，单位为  MWh
- **Token Holders**：平台上  持有货币代币或资产代币  的用户总量。

- **Power Generation Overview**：单位为  美元  的资产代币价格历史，每小时采集代币市价  并记录一次。可选时间单位分别为：24H、7D、30D、3M、6M、1Y、ALL和PRE。24H下显示  从当前小时往前推算24小时  的数据点、ALL下显示  资产代币发行以来  的完整价格变动、PRE显示  生成式AI通过历史价格生成的  未来1个月  价格曲线。所有超过六个月的历史数据  仅保留每一天第23小时的价格（收盘价）。
- **Quick Trade**：Asset栏用输入框代替选择栏，Asset输入框仅能输入26个大写英文字母及小数点标志来指定交易资产代币、Amount输入框仅能输入1~0阿拉伯数字及小数点标志。点击Buy和Sell按钮后弹出框中按view full ticket跳转至股票交易页面。
  
- **Power Station Assets**：显示用户持有的资产代币。发电种类（Hydro或Solar）及发电量取自资产信息。显示价格为  用户持有的该资产代币量*当前的代币市价。此处的显示价格每1分钟采集一次，不记录进档案。百分比增加与减少为  当前显示价格  对比  用户每次购买代币时所支付价格的总合  的百分比。
- **Recent Activity**：用户近期的交易历史，可以显示的交易分成七种：存入货币代币、取出货币代币、购买资产代币、售卖资产代币、获取分红、质押资产代币、解质资产代币。现阶段不实装质押资产代币和解质资产代币。

- **Active Proposals**：现阶段不实装。

## 2. Project页面
- **Filter**：Locations按照实装项目的地点实时更新选项，Capacity、ROI Range和Status如页面所示。
- **右侧快速投资窗口**：随左侧高光选中的项目变更名称和详细信息。
  
- **单个项目详细页面上的数据**：除Next Distribution、Available代币及Current Output取自API外，其他数据均为项目方自己报告。
- **Performance Analytics**：此处显示单位为  MWh  的资产发电历史，与Power Generation Overview的逻辑一样每小时通过API采集斌记录一次。可选时间单位、预测、及历史数据处理方式与Power Generation Overview保持一致。
  
- **点进Distribution Schedule**：项目的季度性分红数据，所有数据均取自项目方实际分红和预估分红。Cash Flow Timeline图标如页面所示。
- **Upcoming Distribution中On-Chain Proof界面**：全部取自链上API。
- **Investor Ledger**：所有详细的项目代币股东分红明细，取自链上数据（链上端同事会负责对接）。
- **Distribution Policy**：其中Insurance Reserve数值取自链上数据。

- **点进Operations Log**：除Audit Trail与Compliance Snapshot需要项目方自行提供设置外，其余皆取自资产方API。

## 3. Trading页面
- **Filter**：Asset Type与Location随资产方种类增加自行更新。Price Range仅支持输入0以上的整数、Yield Range仅支持输入0~100之间的整数。
- **Price Trends**：与Power Generation Overview为同一表格。
- **Trading Volume**：与Power Generation Overview处理方式相似，采集项目为选中代币上一小时的交易总量。
- **点进Power Station Trading Market中详细资产代币后**：Price Chart与Power Generation Overview为同一表格、Transaction History为用户购买该资产代币的历史记录、Order Book为全平台近期成交的买单与卖单。My Orders移出当前子页面，放到Trading主页面上。


## 4. Governance页面
- 现阶段不实装。
  
---
