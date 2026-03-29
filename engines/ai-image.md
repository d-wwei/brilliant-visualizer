# AI 图片生成引擎

从文章上下文生成 AI 图片，支持多后端路由。

## 后端优先级

读取环境变量 `VISUALIZE_AI_BACKEND` (逗号分隔):

```bash
# 默认优先级
VISUALIZE_AI_BACKEND=flux2,gpt-image-1,dall-e-3,nanobanana
```

逐个尝试，第一个可用的后端执行生成。

---

## 后端 1: 本地 Flux2

**配置**:
```bash
VISUALIZE_FLUX2_URL=http://192.168.x.x:port/api/generate
```

**检测**: curl 检查 URL 可达性

```bash
curl -s --max-time 3 "$VISUALIZE_FLUX2_URL" >/dev/null 2>&1 && echo "available" || echo "unavailable"
```

**调用**:
```bash
curl -X POST "$VISUALIZE_FLUX2_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "...",
    "width": 1024,
    "height": 768,
    "num_inference_steps": 28,
    "guidance_scale": 3.5
  }' \
  --output ./images/output.png
```

**特点**: 无费用，速度取决于本地 GPU，风格一致性好。

---

## 后端 2: gpt-image-1

**配置**:
```bash
OPENAI_API_KEY=sk-xxx
```

**调用** (通过 OpenAI API):
```bash
curl -X POST https://api.openai.com/v1/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-image-1",
    "prompt": "...",
    "n": 1,
    "size": "1536x1024",
    "quality": "high"
  }'
```

响应中 `data[0].b64_json` 为 base64 编码的图片，需解码保存:
```bash
echo "$B64_DATA" | base64 -d > ./images/output.png
```

**特点**: 高质量，支持文字渲染，理解复杂场景。

**可用尺寸**: `1024x1024`, `1536x1024` (横向), `1024x1536` (纵向), `auto`

---

## 后端 3: DALL-E 3

**配置**: 同 gpt-image-1，使用 `OPENAI_API_KEY`

**调用**:
```bash
curl -X POST https://api.openai.com/v1/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "dall-e-3",
    "prompt": "...",
    "n": 1,
    "size": "1792x1024",
    "quality": "hd",
    "style": "natural"
  }'
```

响应中 `data[0].url` 为图片 URL，需下载保存:
```bash
curl -o ./images/output.png "$IMAGE_URL"
```

**可用尺寸**: `1024x1024`, `1792x1024` (横向), `1024x1792` (纵向)
**style**: `natural` (真实感) 或 `vivid` (超现实)

---

## 后端 4: nanobanana

**配置**:
```bash
NANOBANANA_API_KEY=xxx
```

**调用**: 根据 nanobanana API 文档，基本模式:
```bash
curl -X POST https://api.nanobanana.com/v1/generate \
  -H "Authorization: Bearer $NANOBANANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "...",
    "width": 1024,
    "height": 768
  }'
```

(具体 API 格式以 nanobanana 官方文档为准，首次使用时确认。)

---

## Prompt 工程

### 从文章上下文构建 Prompt

1. **提取核心概念**: 从文章段落中提取要可视化的关键概念
2. **确定风格**: 根据文章类型和用户指定选择风格
3. **构建描述**: 组合概念 + 风格 + 技术参数

### Prompt 模板

**封面图/题图**:
```
A [风格] illustration representing [文章主题].
[具体视觉元素描述].
Clean composition, professional quality, suitable for article header.
No text, no watermark, no border.
```

**概念可视化**:
```
A [风格] diagram/illustration explaining the concept of [概念名].
Visual metaphor: [隐喻描述].
Simple, clear, educational style.
No text overlays, no watermark.
```

**技术概念图**:
```
A [风格] technical illustration showing [技术概念].
[组件和关系的视觉描述].
Isometric/flat design, tech color palette (blues, purples, teals).
No text, no watermark.
```

### 风格关键词

| 用户指定 | Prompt 关键词 |
|---------|--------------|
| 科技感 | modern tech, digital, futuristic, gradient blues and purples |
| 手绘 | hand-drawn sketch style, pencil lines, warm tones |
| 扁平 | flat design, minimal, bold colors, geometric shapes |
| 3D | 3D rendered, isometric, soft shadows, vibrant colors |
| 照片级 | photorealistic, high resolution, studio lighting |
| 极简 | minimalist, white space, single accent color |
| 商务 | professional, corporate, clean, muted colors |

### 负面 Prompt (所有生成都加)

```
No text, no watermark, no logo, no border, no frame.
No distorted faces, no extra fingers.
```

### 尺寸选择

| 用途 | 比例 | 推荐尺寸 |
|------|------|---------|
| 封面图/题图 | 16:9 | 1792x1024 (DALL-E) / 1536x1024 (gpt-image-1) |
| 正文配图 | 4:3 | 1024x768 |
| 正方形 | 1:1 | 1024x1024 |
| 竖图 | 9:16 | 1024x1792 (DALL-E) / 1024x1536 (gpt-image-1) |

默认: 封面用 16:9，正文用 4:3。

---

## 生成流程

```
1. 从配图方案表读取描述和风格
2. 构建 prompt (模板 + 风格 + 负面 prompt)
3. 选择尺寸 (根据用途)
4. 按优先级尝试后端
5. 保存图片到 ./images/
6. 展示预览
7. 用户确认或重新生成 (可调整 prompt/风格/后端)
```

## 重新生成

用户不满意时:
- 可以修改描述/风格
- 可以指定使用另一个后端
- 可以提供参考图片 URL (仅 gpt-image-1 支持编辑)

## 费用提示

每次 AI 生成会消耗 API 额度:
- gpt-image-1 HD: ~$0.08/张
- DALL-E 3 HD: ~$0.08/张
- 本地 Flux2: 免费
- nanobanana: 按其定价

生成前不需要特别提醒，但如果一篇文章生成 >5 张 AI 图片，提示一次预估费用。
