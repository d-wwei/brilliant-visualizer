# HTML 渲染引擎

将数据对比、信息图、统计图表等生成为 HTML，然后截图保存为图片。

## 工具依赖

任意无头浏览器。Agent 使用其可用的浏览器工具：
- **gstack browse** (Claude Code 内置)
- **Puppeteer** (`npx puppeteer screenshot`)
- **Playwright** (`npx playwright screenshot`)
- 或 Agent 平台自带的浏览器能力

## 工作流程

```
1. 生成 HTML + inline CSS 代码
2. 写入 /tmp/ 临时文件
3. 用无头浏览器打开本地文件
4. 截图保存为 PNG
5. 删除临时文件
```

## 执行步骤

根据 Agent 可用的浏览器工具选择。所有方式都应使用 **3x 缩放**输出高清图片（800px viewport → 2400px 输出）。

**方式 A: gstack browse (Claude Code)**
```bash
$B viewport 800x600 deviceScaleFactor=3
$B goto file:///tmp/visualize-render-{timestamp}.html
$B screenshot "./images/compare-html-20260328-001.png" fullPage
```
> 注意: 必须显式设置 `deviceScaleFactor=3`，不要依赖默认值。

**方式 B: Chrome headless (直接调用)**
```bash
# --force-device-scale-factor=3 输出 3x 高清 (2400x1800)
"$CHROME" --headless=new --disable-gpu \
  --force-device-scale-factor=3 \
  --hide-scrollbars \
  --window-size=800,600 \
  --screenshot="./images/compare-html-20260328-001.png" \
  "file:///tmp/visualize-render-{timestamp}.html"
```

**方式 C: Puppeteer**
```bash
npx puppeteer screenshot file:///tmp/visualize-render-{timestamp}.html \
  --output "./images/compare-html-20260328-001.png" \
  --viewport 800x600 \
  --device-scale-factor 3
```

**方式 D: Playwright**
```bash
npx playwright screenshot file:///tmp/visualize-render-{timestamp}.html \
  "./images/compare-html-20260328-001.png" \
  --viewport-size 800,600 \
  --full-page
```
Playwright 代码中设 `deviceScaleFactor: 3`。

Agent 应根据当前环境自动选择可用工具。

---

## 全局 CSS 规则

所有 HTML 可视化文件**必须**包含以下基础样式。这是硬性约束，不可省略任何一行：

```css
/* === 截图防滚动条 — 必须全部包含 === */
*, *::before, *::after {
  box-sizing: border-box;
  scrollbar-width: none !important;           /* Firefox */
  -ms-overflow-style: none !important;        /* IE/Edge */
}
*::-webkit-scrollbar {
  display: none !important;                   /* Chrome/Safari/Edge */
  width: 0 !important;
  height: 0 !important;
}
html, body {
  margin: 0;
  padding: 0;
  overflow: hidden !important;
}
```

**为什么需要这么多层？**
- `overflow: hidden` 只阻止滚动，但浏览器仍可能渲染滚动条轨道
- `::-webkit-scrollbar { display: none }` 彻底隐藏 Webkit 系滚动条的渲染
- `scrollbar-width: none` 覆盖 Firefox
- `!important` 防止浏览器默认样式覆盖

viewport 高度应留 20% 余量，避免内容紧贴底边触发滚动条。

---

## HTML 模板

### 数据对比表

适用: 新旧方案对比、产品功能对比、方案选型

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
  *, *::before, *::after { box-sizing: border-box; scrollbar-width: none !important; -ms-overflow-style: none !important; }
  *::-webkit-scrollbar { display: none !important; width: 0 !important; height: 0 !important; }
  html, body { margin: 0; padding: 0; overflow: hidden !important; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    background: #fff;
    padding: 40px;
  }
  .title {
    font-size: 24px;
    font-weight: 700;
    color: #1a1a1a;
    margin-bottom: 24px;
    text-align: center;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    font-size: 15px;
  }
  th {
    background: #f0f4f8;
    color: #334155;
    padding: 12px 16px;
    text-align: left;
    font-weight: 600;
    border-bottom: 2px solid #e2e8f0;
  }
  td {
    padding: 12px 16px;
    border-bottom: 1px solid #f1f5f9;
    color: #475569;
  }
  tr:hover td {
    background: #f8fafc;
  }
  .highlight {
    color: #059669;
    font-weight: 600;
  }
  .dim {
    color: #94a3b8;
  }
</style>
</head>
<body>
  <div class="title">方案对比</div>
  <table>
    <thead>
      <tr>
        <th>指标</th>
        <th>方案 A</th>
        <th>方案 B</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>响应时间</td>
        <td class="dim">320ms</td>
        <td class="highlight">45ms</td>
      </tr>
      <!-- 更多行 -->
    </tbody>
  </table>
</body>
</html>
```

### 统计指标卡片

适用: KPI 展示、数据摘要

```html
<style>
  *, *::before, *::after { box-sizing: border-box; scrollbar-width: none !important; -ms-overflow-style: none !important; }
  *::-webkit-scrollbar { display: none !important; width: 0 !important; height: 0 !important; }
  html, body { margin: 0; padding: 0; overflow: hidden !important; }
  .cards {
    display: flex;
    gap: 20px;
    padding: 40px;
    background: #fff;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  .card {
    flex: 1;
    background: #f8fafc;
    border-radius: 12px;
    padding: 24px;
    text-align: center;
  }
  .card .value {
    font-size: 36px;
    font-weight: 700;
    color: #1e293b;
  }
  .card .label {
    font-size: 14px;
    color: #64748b;
    margin-top: 8px;
  }
  .card .change {
    font-size: 13px;
    margin-top: 4px;
  }
  .card .change.up { color: #059669; }
  .card .change.down { color: #dc2626; }
</style>

<div class="cards">
  <div class="card">
    <div class="value">2.4M</div>
    <div class="label">月活用户</div>
    <div class="change up">+12.3%</div>
  </div>
  <div class="card">
    <div class="value">$48.2K</div>
    <div class="label">MRR</div>
    <div class="change up">+8.7%</div>
  </div>
  <div class="card">
    <div class="value">3.2%</div>
    <div class="label">流失率</div>
    <div class="change down">-0.5%</div>
  </div>
</div>
```

### 简单柱状图 (纯 CSS)

适用: 数据对比、占比

```html
<style>
  *, *::before, *::after { box-sizing: border-box; scrollbar-width: none !important; -ms-overflow-style: none !important; }
  *::-webkit-scrollbar { display: none !important; width: 0 !important; height: 0 !important; }
  html, body { margin: 0; padding: 0; overflow: hidden !important; }
  .chart {
    padding: 40px;
    background: #fff;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  .chart .title {
    font-size: 20px;
    font-weight: 600;
    color: #1e293b;
    margin-bottom: 24px;
  }
  .bar-row {
    display: flex;
    align-items: center;
    margin-bottom: 12px;
  }
  .bar-label {
    width: 120px;
    font-size: 14px;
    color: #475569;
  }
  .bar-track {
    flex: 1;
    height: 28px;
    background: #f1f5f9;
    border-radius: 6px;
    overflow: hidden;
  }
  .bar-fill {
    height: 100%;
    border-radius: 6px;
    display: flex;
    align-items: center;
    padding-left: 12px;
    font-size: 13px;
    color: #fff;
    font-weight: 600;
  }
  .bar-fill.blue { background: #3b82f6; }
  .bar-fill.green { background: #10b981; }
  .bar-fill.purple { background: #8b5cf6; }
</style>

<div class="chart">
  <div class="title">市场份额</div>
  <div class="bar-row">
    <div class="bar-label">产品 A</div>
    <div class="bar-track">
      <div class="bar-fill blue" style="width: 65%">65%</div>
    </div>
  </div>
  <div class="bar-row">
    <div class="bar-label">产品 B</div>
    <div class="bar-track">
      <div class="bar-fill green" style="width: 25%">25%</div>
    </div>
  </div>
  <div class="bar-row">
    <div class="bar-label">其他</div>
    <div class="bar-track">
      <div class="bar-fill purple" style="width: 10%">10%</div>
    </div>
  </div>
</div>
```

### 时间线

适用: 项目历程、版本发布

```html
<style>
  *, *::before, *::after { box-sizing: border-box; scrollbar-width: none !important; -ms-overflow-style: none !important; }
  *::-webkit-scrollbar { display: none !important; width: 0 !important; height: 0 !important; }
  html, body { margin: 0; padding: 0; overflow: hidden !important; }
  .timeline {
    padding: 40px;
    background: #fff;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  .timeline .title {
    font-size: 20px;
    font-weight: 600;
    color: #1e293b;
    margin-bottom: 32px;
  }
  .event {
    display: flex;
    margin-bottom: 24px;
  }
  .event .date {
    width: 100px;
    font-size: 13px;
    color: #64748b;
    padding-top: 2px;
  }
  .event .dot-line {
    width: 40px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  .event .dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: #3b82f6;
  }
  .event .line {
    width: 2px;
    flex: 1;
    background: #e2e8f0;
    margin-top: 4px;
  }
  .event .content {
    flex: 1;
  }
  .event .content h4 {
    margin: 0;
    font-size: 15px;
    color: #1e293b;
  }
  .event .content p {
    margin: 4px 0 0;
    font-size: 13px;
    color: #64748b;
  }
</style>

<div class="timeline">
  <div class="title">项目里程碑</div>
  <div class="event">
    <div class="date">2026-01</div>
    <div class="dot-line"><div class="dot"></div><div class="line"></div></div>
    <div class="content">
      <h4>项目启动</h4>
      <p>完成需求分析和技术选型</p>
    </div>
  </div>
  <!-- 更多事件 -->
</div>
```

---

## 生成规则

1. **从文章提取数据**: 解析段落中的数字、对比、列表
2. **选择模板**: 根据数据类型选择对比表/指标卡/柱图/时间线
3. **填充数据**: 将文章数据填入模板
4. **自适应内容**: 如果数据很多，适当增加图片宽度
5. **配色**: 使用中性配色 (蓝灰系)，保持专业感

## 截图参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| viewport 宽度 | 800px | 适合文章配图，宽表格用 1200px |
| viewport 高度 | 内容高度 + 20% | 留余量防止滚动条 |
| deviceScaleFactor | **3** | 输出 3x 高清图片，确保文字锐利清晰 |

3x 缩放意味着 800x600 viewport 输出 2400x1800 的 PNG。文件约 500KB-1MB，文字和细节极其清晰。

各工具设置 3x 的方式：
- **Chrome headless**: `--force-device-scale-factor=3 --hide-scrollbars`
- **gstack browse**: `deviceScaleFactor=3` (必须显式设置，不要依赖默认值)
- **Puppeteer**: `--device-scale-factor 3` 或 `page.setViewport({ deviceScaleFactor: 3 })`
- **Playwright**: `browser.newContext({ deviceScaleFactor: 3 })`

## 注意事项

- HTML 必须自包含 (inline CSS，不依赖外部资源)
- **所有模板必须包含完整的防滚动条 CSS 三件套** (见"全局 CSS 规则"部分)，仅 `overflow: hidden` 不够
- 中文字体: 使用系统字体栈，不引入 web font
- 背景色: 白色，适配各种文章背景
- 宽度: 默认 800px，根据内容调整
- 截图默认 3x 高清，保证 Retina 屏和移动端文字锐利清晰
- Chrome headless 额外加 `--hide-scrollbars` 标志作为双重保险
