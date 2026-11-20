# 基金数据源与实测命令对照表

## 优先级

```
方案 A（首选）：jy-financedata-api（恒生聚源 MCP）
  ↳ 唯一能拿到场外基金专有数据：A/C份额净值序列、同类排名分位、夏普比率、持有人结构
  ↳ 前提：需 JY_API_KEY（https://www.gildata.com/products/datamap申请）

方案 B（降级，实测有效）：westock CLI（内置，无需鉴权）
  ↳ 只覆盖【场内交易品种】；按母ETF / 标的指数口径分析，拿不到场外专有指标

方案 C（2026-11 新增）：天天基金公开 HTTP 接口
  ↳ 场外开放式基金、QDII-LOF、真实净值序列与复权日涨跌；折溢价率查证也走这里
  ↳ 触发条件：westock search 查不到该基金，或需要净值而非场内价格

方案 D（补充）：WebSearch
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

## 方案 B：westock CLI（内置降级，2026-10 实测可用）

> ⚠️ **旧写法已全部失效**（2026-08 记录）：`.../builtin-skills/westock-data/scripts/index.js`
> 该路径在新版本中不存在，且 `etf detail` 已拆分。现为**安装式 CLI** `westock`，直接调用即可。

```bash
command -v westock && westock --help   # 先确认已安装
```

| 目的 | 命令 | 备注 |
|---|---|---|
| 定位代码 | `westock search <名称> --type etf` | 不加 `--type etf` 常查不到基金 |
| **核心档案** | `westock etf profile <code>` | 概况、费率、经理、跟踪指数 |
| **运作概览** | `westock etf overview <code>` | 各周期收益、各周期回撤、估值历史分位 |
| 重仓股 | `westock etf holdings <code>` | **复数 holdings**，前 20 大及权重 |
| 净值序列 | `westock etf nav <code>` | 仍可能取不到，见下方坑表 |
| 实时行情 | `westock quote <code>` | 现价、52周高低、PE、PB、内外盘、各周期涨幅 |
| 趋势 | `westock kline <code> --period month --limit 14` | 月线；`--period year` 取年线 |
| 指数估值 | `westock quote <指数代码>` | 港股指数用 `hk` 前缀，如 `hkHSTECH` |
| 成份股行情 | `westock quote <c1>,<c2>,<c3>` | 逗号分隔批量查 |
| 近期动态 | `westock news <code> --limit 8` | 避免凭空解释涨跌原因 |
| **历史 PE/PB** | `westock quote <codes> --date 2024-12-31` | **历史日行情自带 PE/PB/市值**，可批量算历年估值中位数 |
| **财报利润** | `westock finance <codes> --type income --limit 24` | 归母净利等，见下方硬约束 |

### `westock finance` 硬约束（踩过）

- 批量**一次最多 10 个代码**，超出直接报错
- 有 `LOCAL_RATE_LIMITED` 限流，**分批间隔 20 秒**可通过
- `--limit` 是**期数不是年数**：回溯 5 个年报需 `--limit ≥ 24`（含季报）
- 年报筛选：`EndDate` 以 `12-31` 结尾的行
- 计算 PE 中位数时**必须剔除负 PE（亏损股）**，并标注样本数

### 实测坑（务必先读）

| 坑 | 表现 | 正确做法 |
|---|---|---|
| **联接基金查不到** | `search` 无论加不加 `--type etf` 都返回「未找到」 | 场外份额不在行情库。先确定其**母 ETF**，按母 ETF 分析。例：「汇添富中证主要消费ETF联接A」→ 母ETF `sz159928` 消费ETF汇添富 → 指数 `sh000932` |
| 旧子命令失效 | `etf detail` 无输出或报错 | 已拆为 **`etf profile` / `etf overview` / `etf holdings` / `etf nav`** |
| `etf nav` 取不到 | `执行失败 [DATA_ERROR]: 未找到有效数据` | 改用 `kline --period month` 推演趋势 |
| **返回全 0** | `etf holders` 持有人结构、部分财务字段全是 0 | **这是数据缺失，不是真实值**。标注「⚠️ 数据缺失」，严禁拿 0 做推断 |
| **权重两套口径** | `etf holdings` 同时返回「持仓明细」与「申赎清单」两组权重 | 二者口径不同，**必须二选一并声明用了哪套**，不可混用 |
| **港股列位不同** | 港股返回列顺序与 A 股不一致 | **按表头名解析，绝不按固定列号硬解析** |
| 早年样本偏少 | 批量查历史 PE 时部分公司未上市 | 属正常，需在图注标注每年样本数 |
| 名称搜不中 | 全称搜不到 | 换关键词或直接用 6 位代码搜（如 `westock search 513010`） |
| **裸代码报错** | `etf profile 510310` 报「不支持的…代码」 | `search` 返回 `sh510310` 等**带市场前缀**的 code，原样传下去 |
| **A 股利润字段名** | 套港股/美股教程取不到值 | A 股用 `NPParentCompanyOwners`（年报 12-31 累计）/ `NPParentCompanyOwnersTTM`；美股用 `NetIncome_Q`（季度，百万 USD） |
| **批量上限记混** | 以为 quote 也限 10 只 | `quote <codes> --date` 一次 **30 只可过**；只有 `finance` 限 10 只、需间隔 20 秒 |
| **规模口径不一致** | `profile.totalAsset` 520 亿 vs `quote` 场内市值 1059 亿 | 选定一个口径写明，**不混用、不推断**，差异列入缺失项 |
| **成分股漂移** | 用当前前 30 大回溯 2021 年，结论偏乐观 | 图注标注「按当前权重选取、存在幸存者偏差、仅代表今天的龙头」，不可与全指数口径替代 |
| **净值披露滞后** | 净值到 11-18，行情到 11-20 | 分别标日期，**不做外推** |

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

## 方案 C：天天基金公开接口（2026-11 实测）

所有请求需 `User-Agent`；`pingzhongdata` 用 `Referer: https://fund.eastmoney.com/`，
F10 页面用 `Referer: https://fundf10.eastmoney.com/`。

| 用途 | 端点 | 备注 |
|---|---|---|
| 净值 / 规模 / 经理 / 雷达 | `https://fund.eastmoney.com/pingzhongdata/<code>.js` | 单行压缩 JS，用正则按变量名取值 |
| 费率 / 规模 / 成立日 / 基准 | `https://fundf10.eastmoney.com/jbgk_<code>.html` | HTML，strip tag 后取字段 |
| 阶段涨幅 / 同类排名 | `https://fundf10.eastmoney.com/FundArchivesDatas.aspx?type=jdzf&code=<code>` | 返回 `var apidata={content:"<div>…"` |
| 经理履历 | `https://fundf10.eastmoney.com/jjjl_<code>.html` | 任职回报、任期 |
| 季报 / 半年报持仓 | `...?type=jjcc&code=<code>` | tbody 常为空，用 fund-v2 `fund_evaluator.py` 抓完整文件 |

### `pingzhongdata` 解析

```python
import re, json
s = open("pz.js", encoding="utf-8", errors="ignore").read()
def grab(k):
    m = re.search(k + r'\s*=\s*(\[.*?\]|\{.*?\}|"[^"]*")\s*;', s)  # 外层单引号！
    return json.loads(m.group(1))
nw = grab("Data_netWorthTrend")     # [{x: ms时间戳, y: 单位净值, equityReturn: 复权日涨跌%, unitMoney}]
ac = grab("Data_ACWorthTrend")      # [[ms, 累计净值], ...]
```

⚠️ **坑**：文件是**单行**，不能按行匹配 `^var`；正则外层用**单引号**包裹，
内层含 `"[^"]*"` 双引号才不会截断字符串。`Data_fluctuationScale` 是 **dict**（含 categories + series），
不是 list，别对它做切片。`Data_fundSharesPositions` 可能为占位/估算值，与季报冲突时以季报为准。

**实测返回空的端点**（12 字节，别再试）：`type=zcpz`、`type=hycc`、`type=tsdata`、
`F10DataApi.aspx?type=hycc`。`https://fundgz.1234567.com.cn/js/<code>.js` 对部分 QDII/LOF 返回 404。

### 复权净值重建（必做）

```python
import datetime as dt, statistics as st
def day(ms): return dt.datetime.fromtimestamp(ms/1000, dt.timezone.utc).strftime("%Y-%m-%d")

v, adj = 1.0, []
for r in nw:                                  # 全序列、按时间正序
    v *= (1 + (r.get("equityReturn") or 0) / 100)
    adj.append((day(r["x"]), v))
cur = nw[-1]["y"]                             # 当前单位净值
scale = cur / adj[-1][1]                      # 换算到当前口径（元）
curve = [(d, v*scale) for d, v in adj if d >= "2021-11-20"]
median = st.median([v for _, v in curve])
```

**为什么必须这么做**：510310 在 **2024-09-19** 单位净值 1.5977 → 3.2021（名义 +100.42%），
但当日 `equityReturn` = +0.16% —— 这是**份额折算**；`Data_ACWorthTrend` 仍是折算前口径
（末值 2.1023 < 单位净值 3.9451），两者不同步。直接对单位净值取中位数会混合两种口径。
换算后：中位数 **3.6671 元**，当前 3.9451 元（**+7.58%**），五年区间 2.9121~4.9331 元，1214 个交易日。

### 折溢价率查证

```bash
# ① 判断有无场内份额（看 QuoteID / Classify）
curl -s -H "User-Agent: Mozilla/5.0" \
 "https://searchapi.eastmoney.com/api/suggest/get?input=160213&type=14&token=D43BF722C8E33BDC906FB84D85E326E8&count=15"
# ② 有场内再看行情（以下对 160213 均失败，仅作对照）
curl -s -H "Referer: https://finance.sina.com.cn/" "https://hq.sinajs.cn/list=sz160213"
curl -s "https://qt.gtimg.cn/q=sz160213"
curl -s "https://push2.eastmoney.com/api/qt/stock/get?ut=fa5...&fltt=2&fields=f43,f58,f60&secid=0.160213"
# ③ 集思录 QDII 榜（有 discount_rt，但 rp 参数无效、恒 20 条）
curl -s -X POST -H "Referer: https://www.jisilu.cn/" \
 "https://www.jisilu.cn/data/qdii/qdii_list/E?___jsl=LST___t=<ms>" -d "rp=100&page=1"
```

| 返回 | 含义 |
|---|---|
| `0.<code>` Fund + `150.<code>` OTCFUND 两条 | 真 LOF，有场内价，可算折溢价 |
| 只有 `150.<code>` OTCFUND | 主流行情源无场内报价 → **折溢价率数学上无法计算** |

**实测对照**：161127 有两条登记；160213 只有 OTCFUND 一条 → 新浪返回空串、腾讯静默丢码、
push2 `data:null`（注意 `1.160213` 是债券「20河南05」，同名别混）。
查不到时按 SKILL.md 要求写清查证过程 + 对用户的实际影响，不填估算值。

### 表头名解析（westock 表格通用）

多代码 + `--date` 的响应会**缺少部分列**导致列位漂移，必须按表头名取值：

```python
def parse_md_tables(text):
    blocks, cur = [], []
    for line in text.splitlines():
        s = line.strip()
        if s.startswith("|"): cur.append(s)
        elif cur: blocks.append(cur); cur = []
    if cur: blocks.append(cur)
    out = []
    for b in blocks:
        if len(b) < 2: continue
        hdr = [x.strip() for x in b[0].split("|")][1:-1]
        rows = [dict(zip(hdr, [x.strip() for x in l.split("|")][1:-1]))
                for l in b[2:] if len([x.strip() for x in l.split("|")][1:-1]) == len(hdr)]
        out.append((hdr, rows))
    return out
```

### 方案 C 的能力边界

**拿得到**：真实单位/累计净值序列、复权日涨跌、规模与费率、阶段涨幅与同类排名、经理履历、季报持仓。
**拿不到**：同类排名分位、夏普比率、年化波动率（仍需聚源）；行业配置明细、持有人结构（接口返回空）。
**注意**：净值 T+1 披露，日期常比行情接口晚 1~2 个交易日，**标注日期、不做外推**。

---

## 取数纪律

1. **查后即写**：先记录原始返回，再引用，不凭记忆写数字
2. **交叉校验**：规模、净值、成立日期等关键项，两个来源一致才采用
3. **时间戳**：每个数据标注截止日期（行情日期 ≠ 持仓清单日期，需分别标注）
4. **缺失显式化**：写「⚠️ 数据缺失（原因：需 JY_API_KEY）」，不用 0、不估算、不留白
5. **禁止虚构**：基金经理言论、公司评价必须来自真实查询或搜索结果
6. **口径打架两个都摆**：两个真实口径给出相反结论时，同时呈现 + 解释成因 + 给「正确说法」，
   禁止只挑好看的那个；详见 `case-studies.md`
7. **统计前先看完整输出**：禁止用 `sed -n '6,60p'` / `head` 截断后再统计只数或合计
   （曾因此把 100 只持仓误判为 52 只）
