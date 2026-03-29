# Brilliant Visualizer — 文章智能配图 Skill 设计文档

**日期**: 2026-03-28
**作者**: Eli + Claude
**状态**: Draft

---

## 1. 概述

Brilliant Visualizer 是一个文章智能配图 skill，分析文章内容后自动识别需要配图的位置，选择最佳图片引擎，生成或搜索配图后插入文章。

**定位**: 写作工具链的中间环节，衔接 great-writer（写作）和 typeset（排版）。

**核心理念**: 两步走 — 先分析推荐，用户确认后再生成。

## 2. 基本信息

- **Skill 名**: `brilliant-visualizer`
- **触发命令**: `/visualize`
- **触发词**: "配图", "illustrate", "给文章加图", "插图", "add illustrations", "visualize"
- **目录**: `~/.claude/skills/brilliant-visualizer/`

## 3. 架构: 轻量路由 + 按需加载

```
~/.claude/skills/brilliant-visualizer/
  SKILL.md              ← 主入口 (~200-300 行): 分析器 + 路由逻辑 + 工作流
  engines/
    mermaid.md          ← Mermaid 图表生成规则 + 常用模板
    architecture.md     ← D2/Graphviz 架构图规则 + 模板
    ai-image.md         ← AI 图片生成 (prompt 工程 + 多后端路由)
    stock-photo.md      ← 图库搜索策略 + fallback 到 web search
    html-render.md      ← HTML→截图 (信息图/对比图/数据可视化)
```

**按需加载**: SKILL.md 只包含路由逻辑，具体引擎规则在需要时才读取对应 `engines/*.md` 文件。

## 4. 工作流

### 4.1 阶段一: 分析 (Analyze)

**输入**: 文章 Markdown 文件路径，或剪贴板/粘贴的 Markdown 内容

**分析维度**:
1. 扫描文章结构 (标题、段落、列表、代码块)
2. 识别内容类型 (流程描述、架构说明、数据对比、概念解释、场景叙述)
3. 判断配图位置 (章节转折点、复杂概念处、数据密集处、文章首尾)
4. 选择最佳引擎 (根据内容类型匹配)
5. 生成配图描述 (为每个配图点写具体的图片描述)

**输出**: 配图方案表

```markdown
## 配图方案

| # | 位置 | 类型 | 引擎 | 描述 | 优先级 |
|---|------|------|------|------|--------|
| 1 | 标题后 | 封面图 | ai-image | 文章主题概念图，科技感 | 必要 |
| 2 | §2 "系统架构" 后 | 架构图 | mermaid | 微服务拓扑，含 3 个服务 | 必要 |
| 3 | §3 "用户流程" 后 | 流程图 | mermaid | 注册→激活→首次使用 | 必要 |
| 4 | §5 "性能对比" 后 | 对比图 | html-render | 新旧方案性能指标对比 | 建议 |
| 5 | §6 结尾 | 氛围图 | stock-photo | 团队协作场景 | 可选 |
```

**用户交互**: 用户可以确认全部、修改引擎/描述、删除不需要的、新增遗漏的。

### 4.2 阶段二: 生成 (Generate)

按确认后的方案逐个生成:

1. **环境检查**: 检测所需引擎工具是否安装，未安装则提示安装
2. **按优先级生成**: 必要 > 建议 > 可选
3. **输出处理**:
   - 图表类 (Mermaid/D2) → 内联代码块插入 Markdown
   - 图片类 (AI生成/图库) → 保存到 `./images/` 目录，Markdown 中插入 `![desc](images/xxx.png)`
4. **逐个确认**: 每生成一个，展示预览让用户确认或重新生成

## 5. 引擎路由规则

### 5.1 内容类型→引擎映射

| 内容类型 | 关键词/模式 | 引擎 |
|---------|------------|------|
| 流程/步骤/决策 | "流程", "步骤", "如果...则...", 有序列表 | mermaid (flowchart) |
| 时间序列交互 | "请求", "响应", "调用", API 描述 | mermaid (sequence) |
| 实体关系 | "数据模型", "表结构", "一对多" | mermaid (erDiagram) |
| 状态变迁 | "状态", "生命周期", "转换" | mermaid (stateDiagram) |
| 时间规划 | "里程碑", "计划", "时间线" | mermaid (gantt) |
| 层级结构 | "分类", "层次", "树形" | mermaid (mindmap) |
| 系统架构 | "架构", "组件", "模块", "部署" | architecture (D2 优先, fallback Mermaid) |
| 数据对比 | "对比", "指标", "性能", 表格数据 | html-render |
| 抽象概念 | 需要隐喻或视觉化的概念 | ai-image |
| 真实场景 | 人物、环境、产品实物 | stock-photo |
| 封面/题图 | 文章标题 | ai-image (优先) 或 stock-photo |

### 5.2 引擎优先级回退

每个引擎有 fallback 链:
- architecture: D2 → Graphviz → Mermaid (graph)
- stock-photo: Unsplash API → Pexels API → Web Search
- ai-image: 本地 Flux2 → gpt-image-1 → DALL-E 3 → nanobanana
- html-render: gstack browse 截图

## 6. 引擎详细设计

### 6.1 Mermaid 引擎 (`engines/mermaid.md`)

**工具依赖**: `@mermaid-js/mermaid-cli` (mmdc)
**安装**: `npm install -g @mermaid-js/mermaid-cli`

**支持的图表类型**:
- flowchart / graph (流程图)
- sequenceDiagram (时序图)
- classDiagram (类图)
- erDiagram (ER 图)
- stateDiagram-v2 (状态图)
- gantt (甘特图)
- mindmap (思维导图)
- pie (饼图)

**输出模式**:
- 默认: 内联 Mermaid 代码块 (```mermaid ... ```)
- 可选: 渲染为 PNG/SVG 文件 (当目标平台不支持 Mermaid 渲染时)

**模板**: 为每种图表类型提供常用模板和最佳实践。

### 6.2 架构图引擎 (`engines/architecture.md`)

**工具依赖**: D2 (`d2`) 或 Graphviz (`dot`)
**安装**: `brew install d2` 或 `brew install graphviz`

**使用场景**: 系统架构、网络拓扑、组件依赖、部署图

**输出**: SVG/PNG 文件，保存到 `./images/`

**风格**: 提供 2-3 种预设风格 (简洁线框、彩色分组、手绘风)

### 6.3 AI 图片引擎 (`engines/ai-image.md`)

**后端路由** (按配置优先级):

| 后端 | 调用方式 | 特点 |
|------|---------|------|
| 本地 Flux2 | HTTP API (局域网) | 最快、无费用、风格可控 |
| gpt-image-1 | OpenAI API | 高质量、支持文字渲染 |
| DALL-E 3 | OpenAI API | 成熟稳定 |
| nanobanana | nanobanana API | 备选 |

**配置**: 通过环境变量

```bash
VISUALIZE_AI_BACKEND=flux2,gpt-image-1,dall-e-3,nanobanana  # 优先级顺序
VISUALIZE_FLUX2_URL=http://192.168.x.x:port/api/generate     # 本地 Flux2 地址
OPENAI_API_KEY=sk-xxx                                         # OpenAI key
NANOBANANA_API_KEY=xxx                                        # nanobanana key
```

**Prompt 工程**:
- 根据文章上下文生成描述性 prompt
- 支持风格指定: 科技感、手绘、扁平、3D、照片级
- 自动添加负面 prompt (no text, no watermark, etc.)
- 尺寸自适应: 封面图 16:9, 正文配图 4:3 或 1:1

**输出**: PNG/WebP 文件，保存到 `./images/`

### 6.4 图库搜索引擎 (`engines/stock-photo.md`)

**API 路由**:
1. Unsplash API (需要 API key, 50 req/hour 免费)
2. Pexels API (需要 API key, 200 req/month 免费)
3. Fallback: Web Search (Tavily/WebSearch)

**搜索策略**:
- 从文章上下文提取关键词 (中文→英文翻译)
- 多关键词组合搜索
- 按相关度、质量、尺寸排序
- 返回 Top 3 候选让用户选择

**配置**:
```bash
UNSPLASH_ACCESS_KEY=xxx
PEXELS_API_KEY=xxx
```

**输出**: 下载图片到 `./images/`, 附带图片来源和 license 信息

### 6.5 HTML 渲染引擎 (`engines/html-render.md`)

**工具依赖**: gstack browse (headless Chromium)

**使用场景**:
- 数据对比表格 (带样式)
- 信息图 (统计数据可视化)
- 对比矩阵 (产品/方案对比)
- 简单图表 (用 CSS/SVG 绘制)

**工作流程**:
1. 生成 HTML + inline CSS 代码
2. 用 gstack browse 打开本地 HTML 文件
3. 截图保存为 PNG

**输出**: PNG 文件，保存到 `./images/`

## 7. 环境检查与自动安装

Skill 启动时检查工具可用性:

```bash
# 检查各工具是否安装
command -v mmdc     # Mermaid CLI
command -v d2       # D2
command -v dot      # Graphviz
```

未安装的工具: 提示用户安装命令，不自动安装。

AI 后端: 检查环境变量是否配置，未配置的后端自动跳过。

## 8. 输出格式

### 8.1 混合模式

| 图片类型 | 输出方式 |
|---------|---------|
| Mermaid 图表 | 内联 ```mermaid 代码块 (目标平台支持时) |
| Mermaid 图表 | 渲染为 PNG + `![](images/xxx.png)` (目标平台不支持时) |
| D2/Graphviz | 渲染为 SVG/PNG + `![](images/xxx.png)` |
| AI 生成图 | PNG + `![](images/xxx.png)` |
| 图库图片 | 下载 PNG/JPG + `![](images/xxx.png)` + 来源注释 |
| HTML 渲染 | 截图 PNG + `![](images/xxx.png)` |

### 8.2 图片目录结构

```
article-directory/
  article.md
  images/
    cover-ai-20260328-001.png
    arch-mermaid-20260328-002.svg
    flow-mermaid-20260328-003.png
    compare-html-20260328-004.png
    scene-unsplash-20260328-005.jpg
```

命名规则: `{类型}-{引擎}-{日期}-{序号}.{ext}`

### 8.3 图片来源标注

图库图片自动添加来源注释:
```markdown
![团队协作场景](images/scene-unsplash-20260328-005.jpg)
<!-- Photo by John Doe on Unsplash, License: Unsplash License -->
```

## 9. 与 great-writer 集成

### 9.1 作为 great-writer 管线的可选阶段

```
Research → Draft → [Visualize] → Review → Humanize → Finalize
```

great-writer 在 Draft 完成后，检测到文章可能需要配图时，提示:
"文章已完成初稿。要调用 /visualize 配图吗？"

### 9.2 独立调用

```
/visualize article.md           # 分析并配图
/visualize --analyze-only       # 仅分析，不生成
/visualize --engine mermaid     # 只生成 Mermaid 图表
/visualize --engine ai-image    # 只生成 AI 图片
```

### 9.3 与 typeset 的衔接

- 配图完成后的 Markdown 可以直接交给 typeset 排版
- Mermaid 代码块: typeset 已支持渲染
- 图片文件: typeset 使用 `![](path)` 引用

## 10. 配置汇总

所有配置通过环境变量管理:

```bash
# AI 图片后端 (逗号分隔，按优先级)
VISUALIZE_AI_BACKEND=flux2,gpt-image-1,dall-e-3,nanobanana

# 本地 Flux2
VISUALIZE_FLUX2_URL=http://192.168.x.x:port/api/generate

# OpenAI (DALL-E 3 + gpt-image-1)
OPENAI_API_KEY=sk-xxx

# nanobanana
NANOBANANA_API_KEY=xxx

# 图库
UNSPLASH_ACCESS_KEY=xxx
PEXELS_API_KEY=xxx

# 图片输出目录 (默认 ./images/)
VISUALIZE_OUTPUT_DIR=./images

# Mermaid 输出模式 (inline | render)
VISUALIZE_MERMAID_MODE=inline
```

## 11. 非目标 (YAGNI)

以下功能不在第一版范围内:
- 图片编辑/裁剪/滤镜
- 图片压缩/优化
- 自动上传到图床
- 动画/GIF 生成
- LaTeX 公式渲染 (由 typeset 处理)
- 视频截图/嵌入
