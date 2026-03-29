# HTML 渲染引擎

将数据对比、信息图、统计图表等生成为 HTML，然后截图保存为图片。

## 工具依赖

gstack browse (headless Chromium)。无需额外安装。

## 工作流程

```
1. 生成 HTML + inline CSS 代码
2. 写入 /tmp/ 临时文件
3. gstack browse 打开本地文件
4. 截图保存为 PNG
5. 删除临时文件
```

## 执行步骤

```bash
# 1. 生成 HTML 文件 (由 AI 根据文章数据生成)
# 写入 /tmp/visualize-render-{timestamp}.html

# 2. 用 gstack browse 截图
$B goto file:///tmp/visualize-render-{timestamp}.html
$B screenshot "./images/compare-html-20260328-001.png"

# 3. 清理
rm /tmp/visualize-render-{timestamp}.html
```

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
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    background: #fff;
    padding: 40px;
    margin: 0;
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

```bash
# 默认视口: 800px 宽 (适合文章配图)
$B viewport 800x600

# 如果内容宽度需要更大
$B viewport 1200x800

# 截图 (自动裁剪空白)
$B screenshot "./images/compare-html-20260328-001.png"
```

## 注意事项

- HTML 必须自包含 (inline CSS，不依赖外部资源)
- 中文字体: 使用系统字体栈，不引入 web font
- 背景色: 白色，适配各种文章背景
- 宽度: 默认 800px，根据内容调整
