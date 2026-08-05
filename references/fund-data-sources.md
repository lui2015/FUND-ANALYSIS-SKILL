# 基金数据源与实测命令对照表

## 优先级

```
方案 A（首选）：jy-financedata-api（恒生聚源 MCP）
  ↳ 唯一能拿到场外基金专有数据：A/C份额净值序列、同类排名分位、夏普比率、持有人结构
  ↳ 前提：需 JY_API_KEY（https://www.gildata.com/products/datamap申请）

方案 B（降级，实测有效）：westock-data（内置，无需鉴权）
  ↳ 按母ETF / 标的指数口径分析，拿不到场外专有指标

方案 C（补充）：WebSearch
  ↳ 仅用于基金经理访谈、公司新闻、行业观点等非结构化信息，不用于取数
```

---

## 方案 A：聚源 MCP

### 配置

```bash
mcporter config add jy-financedata-api  --url "https://api.gildata.com/mcp-servers/aidata-assistant-srv-api?token=<KEY>"
mcporter config add jy-financedata-tool --url "https://api.gildata.com/mcp-servers/aidata-assistant-srv-tool?token=<KEY>"
mcporter list | grep jy-finance   # 验证
```

### 工具对照

| 维度 | 工具名 | 拿到什么 |
|---|---|---|
| 概况 | `FundBasicInfoReport` | 代码、名称、类型、规模、成立日、费率 |
| 业绩 | `StageIncreaseReport` | 各周期收益率 + **同类排名**（方案 B拿不到） |
| 风险 | `FundIncomeRiskReport` | **夏普比率、波动率、回撤**（方案 B 拿不到） |
| 经理 | `FundHistoryManagerReport` | 现任/历任履历、管理业绩 |
| 配置 | `FOFAssetDistReport` | 股票/债券/现金仓位 |
| 持仓 | `ShareholdingDetailReport` | 重仓股与集中度 |
| 公司 | `FundCompanyBasicInfo` | 公司规模、投研团队、财务 |

调用格式：
```bash
mcporter call jy-financedata-api.<工具名> "query=基金<代码>"
```

⚠️ **该服务没有通用 `FinancialDataAPI` 工具**，必须用上表具体工具名。

### 批量脚本

```bash
FUND_COMPANY=<基金公司名> python3 ~/.workbuddy/skills/jy-fund-research-report-v2/scripts/fetch_data.py <代码>
# 输出：~/.workbuddy/skills/jy-fund-research-report-v2/output/<代码>_data.md
# 注意：company 维度按公司名查询，不传 FUND_COMPANY 会默认查「易方达基金」导致取错数据
```

---

## 方案 B：westock-data（内置降级）

```bash
W=/Applications/WorkBuddy.app/Contents/Resources/app.asar.unpacked/resources/builtin-skills/westock-data
N=node   # 如用 WorkBuddy 托管 Node，替换为 ~/.workbuddy/binaries/node/versions/<版本>/bin/node
cd $W
```

| 目的 | 命令 | 备注 |
|---|---|---|
| 定位代码 | `$N scripts/index.js search <名称> --type etf` | 不加 `--type etf` 查不到基金 |
| **核心档案** | `$N scripts/index.js etf detail <code>` | **信息量最大**：概况+费率+各周期收益+各周期回撤+基金经理完整履历+跟踪指数+折溢价 |
| 重仓股 | `$N scripts/index.js etf holdings <code>` | **复数holdings**，前 20 大及权重 |
| 实时行情 | `$N scripts/index.js quote <code>` | 现价、52周高低、PE、PB、内外盘、各周期涨幅 |
| 趋势 | `$N scripts/index.js kline <code> --period month --limit 14` | 月线 |
| 指数估值 | `$N scripts/index.js quote <指数代码>` | 指数 PE |
| 成份股行情 | `$N scripts/index.js quote <c1>,<c2>,<c3>` | 逗号分隔批量查 |

### 实测坑（务必先读）

| 坑 | 表现 | 正确做法 |
|---|---|---|
| **联接基金查不到** | `search` 无论加不加 `--type etf` 都返回「未找到」 | 场外份额不在行情库。先确定其**母 ETF**，按母 ETF 分析。例：「汇添富中证主要消费ETF联接A」→ 母ETF `sz159928` 消费ETF汇添富 → 指数 `sh000932` |
| `etf holding` 报错 | `错误: 未知子命令 "holding"` | 用 **`etf holdings`**（复数） |
| `etf nav` 取不到 | `执行失败 [DATA_ERROR]: 未找到有效数据` | 改用 `kline --period month` 推演趋势 |
| **返回全 0** | `etf holders` 持有人结构、部分财务字段全是 0 | **这是数据缺失，不是真实值**。标注「⚠️ 数据缺失」，严禁拿 0 做推断 |
| 名称搜不中 | 全称搜不到 | 换关键词或直接用 6 位代码搜（如 `search 159928 --type etf`） |

### 方案 B 的能力边界（报告中必须声明）

**拿得到**：概况、费率、各周期收益与回撤、重仓股权重、实时行情、52周分位、PE/PB、基金经理履历、跟踪指数与折溢价

**拿不到**（需方案 A）：
- 同类排名分位
- 夏普比率、年化波动率、Beta、信息比率
- A/C 份额各自的净值序列与规模
- 持有人结构（个人/机构占比）
- 基金公司管理总规模与财务数据

**口径偏差**：联接基金因保留现金应对申赎、存在申赎确认时滞，**实际净值通常略低于母 ETF**，年化差异一般 0.1%~0.5%。必须在报告顶部警示条中说明。

---

## 取数纪律

1. **查后即写**：先记录原始返回，再引用，不凭记忆写数字
2. **交叉校验**：规模、净值、成立日期等关键项，两个来源一致才采用
3. **时间戳**：每个数据标注截止日期（行情日期 ≠ 持仓清单日期，需分别标注）
4. **缺失显式化**：写「⚠️ 数据缺失（原因：需 JY_API_KEY）」，不用 0、不估算、不留白
5. **禁止虚构**：基金经理言论、公司评价必须来自真实查询或搜索结果
