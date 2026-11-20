# HTML 报告模板参考

单文件、全内联、浅色主题、离线可看。直接复制骨架后替换 `{{...}}` 占位内容。

## 硬性规范

- **浅色主题**：背景 `#f7f8fa`，卡片 `#fff`，正文 `#1f2328`，次要文字 `#5b6470`
- **中国股市颜色**：涨 `#e63946`（红）/ 跌 `#2a9d8f`（绿）—— **绝不可反**
- 货币符号 `¥`
- 字体：`-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif`
- 容器 `max-width: 960px; margin: 0 auto`
- 图表**用纯内联 SVG**，不引外部 JS，保证断网可看
- 术语解释块：浅黄底 `#fff8e1` + 左边框`#f0b429`

## 骨架

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{{基金简称}} 价值分析报告</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: -apple-system, "PingFang SC", "Microsoft YaHei", sans-serif;
         background: #f7f8fa; color: #1f2328; line-height: 1.75; padding: 24px 16px; }
  .wrap { max-width: 960px; margin: 0 auto; }
  .card { background: #fff; border-radius: 12px; box-shadow: 0 2px 12px rgba(0,0,0,.06);
          padding: 24px; margin-bottom: 20px; }
  h1 { font-size: 26px; margin-bottom: 8px; }
  h2 { font-size: 20px; margin-bottom: 16px; padding-left: 12px;
       border-left: 4px solid #2563eb; }
  h3 { font-size: 16px; margin: 14px 0 8px; }
  .sub { color: #5b6470; font-size: 14px; }
  .up { color: #e63946; font-weight: 600; }    /* 涨=红 */
  .down { color: #2a9d8f; font-weight: 600; }  /* 跌=绿 */
  .warn-bar { background: #fff4e5; border: 1px solid #f0b429; border-radius: 8px;
              padding: 14px 16px; margin-bottom: 20px; font-size: 14px; }
  .term { background: #fff8e1; border-left: 3px solid #f0b429; border-radius: 6px;
          padding: 10px 14px; margin: 10px 0; font-size: 14px; color: #6b5400; }
  .quote { border-left: 3px solid #94a3b8; padding-left: 14px; margin: 12px 0;
           font-style: italic; color: #475569; }
  .grid2 { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px,1fr)); gap: 16px; }
  .metric { background: #f9fafb; border-radius: 8px; padding: 14px; }
  .metric .k { font-size: 13px; color: #5b6470; }
  .metric .v { font-size: 22px; font-weight: 700; margin-top: 4px; }
  table { width: 100%; border-collapse: collapse; font-size: 14px; margin: 12px 0; }
  th, td { padding: 10px 12px; text-align: left; border-bottom: 1px solid #eceff3; }
  th { background: #f4f6f8; font-weight: 600; }
  ul { padding-left: 22px; } li { margin: 6px 0; }
  .foot { color: #7b8494; font-size: 12px; text-align: center; padding: 20px 0; }
  @media (max-width: 600px) { h1 { font-size: 21px; } .card { padding: 18px; } }
</style>
</head>
<body>
<div class="wrap">

  <!-- ①顶部：名称 + 评分环 + 一句话结论 -->
  <div class="card">
    <h1>{{基金全称}}</h1>
    <p class="sub">代码 {{code}}ㆍ数据截至 {{date}}ㆍ标的指数 {{index}}</p>
    <!-- 评分环：dasharray = 周长(2πr≈339, r=54) × 得分/100 -->
    <svg width="140" height="140" viewBox="0 0 140 140">
      <circle cx="70" cy="70" r="54" fill="none" stroke="#eceff3" stroke-width="12"/>
      <circle cx="70" cy="70" r="54" fill="none" stroke="{{色值}}" stroke-width="12"
              stroke-linecap="round" stroke-dasharray="{{339*得分/100}} 339"
              transform="rotate(-90 70 70)"/>
      <text x="70" y="72" text-anchor="middle" font-size="34" font-weight="700"
            fill="#1f2328">{{得分}}</text>
      <text x="70" y="94" text-anchor="middle" font-size="12" fill="#5b6470">综合评分</text>
    </svg>
    <p><strong>{{一句话结论}}</strong></p>
  </div>

  <!-- 口径警示条：仅降级口径时出现 -->
  <div class="warn-bar">⚠️ <strong>数据口径说明</strong>：{{说明替代口径与缺失项}}</div>

  <!-- ② 先说结论 -->
  <div class="card">
    <h2>先说结论</h2>
    <ul>{{3~5 条大白话要点，关键处加 <strong>}}</ul>
  </div>

  <!-- ③ 巴菲特四把尺子 -->
  <div class="card">
    <h2>巴菲特的四把尺子</h2>
    <div class="grid2">
      <!-- 每张卡：标题+评级emoji / 一句话结论 / 数据 / 巴菲特会怎么说 -->
      <div class="metric">
        <h3>能力圈 ✅</h3>
        <p>{{结论}}</p>
        <p class="sub">{{数据支撑}}</p>
        <div class="quote">{{巴菲特原话/点评}}</div>
      </div>
      <!-- 护城河 / 安全边际 / 长期主义 同结构 -->
    </div>
  </div>

  <!-- ④ 芒格三件武器 -->
  <div class="card">
    <h2>芒格的三件武器</h2>
    <h3>反过来想：最惨会怎样</h3>
    <p>{{最坏情况推演}}</p>
    <div class="term">💡 {{回本算法等术语解释}}</div>
    <h3>多元思维模型</h3>
    <p>{{不同板块用不同逻辑}}</p>
    <h3>避免愚蠢</h3>
    <p>{{买这只基金的人通常怎么亏的}}</p>
  </div>

  <!-- ⑤ 看图说话 -->
  <div class="card">
    <h2>看图说话</h2>
    <!-- 走势折线：polyline points="x,y ..."，y 需按 (max-v)/(max-min)*h 归一化 -->
    <!-- 重仓股条形：每行 rect width 按权重比例，fill="#2563eb" -->
    <!-- 各周期收益柱：正值fill="#e63946"，负值 fill="#2a9d8f" -->
  </div>

  <!-- ⑥ 适合谁 -->
  <div class="card">
    <h2>这基金适合谁</h2>
    <table>
      <tr><th>人群</th><th>适配</th><th>说明</th></tr>
      {{✅适合 / ⚠️谨慎 / ❌不适合 各若干行}}
    </table>
  </div>

  <!-- ⑦ 底部 -->
  <div class="card">
    <h2>数据来源与缺失项</h2>
    <p class="sub">来源：{{...}}ㆍ时间戳：{{...}}</p>
    <ul>{{缺失项清单及原因}}</ul>
  </div>

  <div class="foot">
    本报告由 AI 基于公开数据生成，仅供参考，不构成任何投资建议。<br>
    基金投资有风险，过往业绩不代表未来表现。投资者应根据自身风险承受能力独立决策。<br>
    数据可能存在延迟或误差，请以基金公司官方披露为准。
  </div>

</div>
</body>
</html>
```

## SVG 绘图速查

**折线图**（价格走势）
```
可用高度 h=200，宽 w=880，留左边距 40
x_i = 40 + i * (w-40) / (n-1)
y_i = 10 + (maxV - v_i) / (maxV - minV) * h
<polyline fill="none" stroke="#2563eb" stroke-width="2" points="x0,y0 x1,y1 ..."/>
```

**横向条形图**（重仓股权重）
```
每行 y = i*26，条长 = weight / maxWeight * 600
<rect x="120" y="{{y}}" width="{{len}}" height="16" rx="3" fill="#2563eb"/>
<text x="0" y="{{y+13}}" font-size="13">{{股票名}}</text>
<text x="{{125+len}}" y="{{y+13}}" font-size="12" fill="#5b6470">{{weight}}%</text>
```

**涨跌柱**（各周期收益，零轴居中）
```
零轴 y0 = 100；正值向上画（红 #e63946），负值向下画（绿 #2a9d8f）
高度 = |pct| / maxAbs * 80
正值：<rect y="{{y0-h}}" height="{{h}}" fill="#e63946"/>
负值：<rect y="{{y0}}"   height="{{h}}" fill="#2a9d8f"/>
```

**净值趋势折线 + 五年中位数虚线**（2026-11 新增，第四张图，880×280）
```
y 轴按 [min, max] 留一点余量；x 按【日期距起点的天数比例】定位，不要按点索引
<line x1=x0 y1=Y(median) x2=x1 y2=Y(median) stroke="#c5a569" stroke-dasharray="7 5"/>
<text 右对齐>五年净值中位数 {{median}} 元</text>
折线：<path d="M x0 y0 L x1 y1 ..." stroke="#3d6fb4"/>
标注：当前点（红）、五年最高（蓝）、五年最低（绿）、样本天数与起止日期
有份额折算：画竖虚线 + 「单位净值口径切换（复权曲线无跳变）」
```

⚠️ 采样（如每 3 个交易日取一点）时**必须把真实最高/最低点强制加入采样点**，
否则标注数值与图上点位对不上。曲线换算到当前口径（元），才能与现价对照。

## 自检清单（生成后逐条过）

- [ ] 涨红跌绿没搞反
- [ ] 每个术语首次出现都有 `.term` 大白话解释
- [ ] 没有出现禁用词（业绩承压/静待花开/逢低布局/震荡蓄势）
- [ ] 降级口径已在顶部 `.warn-bar` 显著声明
- [ ] 缺失数据标注为「⚠️ 数据缺失+原因」，没有用 0 冒充
- [ ] 评分环 dasharray 数值与得分一致
- [ ] 无外部 CSS/JS 依赖，断网可正常打开
- [ ] 手机宽度下不横向溢出
- [ ] 底部免责声明完整
- [ ] **HTML 标签配平**（div/section/table/p/span/b/tr/td 开闭数相等）
- [ ] **SVG 坐标落在 viewBox 内**，长标注没有被裁切
- [ ] **图上标注的极值与曲线采样点严格一致**
- [ ] **口径打架时两个结论都摆出来了**，并给了「正确说法」
- [ ] 不同接口的数据分别标了日期（净值 T+1 ≠ 行情当日），没有外推
- [ ] 用浏览器对**单个图表元素**截图核对过（长页面整图缩放后会误判）
