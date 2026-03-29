# 图库搜索引擎

从免费图库搜索匹配文章内容的高质量图片。

## 搜索优先级

```
Unsplash API → Pexels API → Web Search (fallback)
```

没有 API key 时自动跳到下一个。全部没有 → Web Search 兜底。

---

## Unsplash API

**配置**:
```bash
UNSPLASH_ACCESS_KEY=xxx
```

**注册**: https://unsplash.com/developers (免费，50 requests/hour)

**搜索**:
```bash
curl -s "https://api.unsplash.com/search/photos?query=KEYWORDS&per_page=5&orientation=landscape" \
  -H "Authorization: Client-ID $UNSPLASH_ACCESS_KEY"
```

**响应解析**:
```json
{
  "results": [
    {
      "id": "xxx",
      "urls": {
        "regular": "https://images.unsplash.com/photo-xxx?w=1080",
        "full": "https://images.unsplash.com/photo-xxx"
      },
      "user": {
        "name": "John Doe"
      },
      "links": {
        "download_location": "https://api.unsplash.com/photos/xxx/download"
      }
    }
  ]
}
```

**下载** (必须触发 download endpoint 以遵守 API 规则):
```bash
# 1. 触发下载统计
curl -s "$DOWNLOAD_LOCATION" -H "Authorization: Client-ID $UNSPLASH_ACCESS_KEY"

# 2. 下载图片
curl -o ./images/scene-unsplash-20260328-001.jpg "$REGULAR_URL"
```

**来源标注** (Unsplash 要求):
```markdown
![描述](images/scene-unsplash-20260328-001.jpg)
<!-- Photo by John Doe on Unsplash -->
```

---

## Pexels API

**配置**:
```bash
PEXELS_API_KEY=xxx
```

**注册**: https://www.pexels.com/api/ (免费，200 requests/month)

**搜索**:
```bash
curl -s "https://api.pexels.com/v1/search?query=KEYWORDS&per_page=5&orientation=landscape" \
  -H "Authorization: $PEXELS_API_KEY"
```

**响应解析**:
```json
{
  "photos": [
    {
      "id": 123,
      "src": {
        "large": "https://images.pexels.com/photos/123/xxx.jpeg?w=1200",
        "original": "https://images.pexels.com/photos/123/xxx.jpeg"
      },
      "photographer": "Jane Smith",
      "photographer_url": "https://www.pexels.com/@janesmith"
    }
  ]
}
```

**下载**:
```bash
curl -o ./images/scene-pexels-20260328-001.jpg "$LARGE_URL"
```

**来源标注** (Pexels 要求):
```markdown
![描述](images/scene-pexels-20260328-001.jpg)
<!-- Photo by Jane Smith on Pexels -->
```

---

## Web Search Fallback

没有图库 API key 时，使用 Web Search 搜索图片:

1. 使用 WebSearch 工具搜索 `"KEYWORDS" site:unsplash.com OR site:pexels.com`
2. 从搜索结果提取图片页面 URL
3. 使用 WebFetch 获取页面，提取图片直链
4. 下载图片

或者使用已安装的搜索工具 (Tavily 等) 搜索免费可商用图片。

**注意**: Web Search fallback 速度较慢，图片质量不如 API 直接搜索可控。

---

## 搜索策略

### 关键词提取

从文章段落中提取搜索关键词:

1. **核心主题词**: 文章标题和段落的核心名词
2. **场景词**: 描述环境、动作、氛围的词
3. **中→英翻译**: 中文关键词自动翻译为英文搜索 (英文图库覆盖更广)

```
文章段落: "团队协作是敏捷开发的核心"
提取关键词: team collaboration, agile development
搜索查询: "team collaboration agile software"
```

### 多轮搜索

如果第一轮搜索结果不理想:
1. 放宽关键词 (去掉限定词)
2. 换同义词
3. 尝试更抽象的描述

### 候选展示

返回 Top 3 候选图片，附预览信息:
```
候选 1: "团队在白板前讨论" — 1920x1280, Unsplash, by John Doe
候选 2: "办公室协作场景" — 2400x1600, Pexels, by Jane Smith
候选 3: "远程会议屏幕" — 1800x1200, Unsplash, by Bob Lee

选择哪张？(1/2/3/重新搜索)
```

---

## 图片尺寸选择

| 用途 | orientation 参数 | 最小宽度 |
|------|-----------------|---------|
| 封面图 | landscape | 1200px |
| 正文配图 | landscape | 800px |
| 侧边配图 | portrait 或 squarish | 600px |

默认用 landscape (横向)，除非用户指定竖图。

## License 说明

| 图库 | License | 可商用 | 要求标注 |
|------|---------|--------|---------|
| Unsplash | Unsplash License | 是 | 建议标注，不强制 |
| Pexels | Pexels License | 是 | 建议标注，不强制 |

两个图库都免费商用，但好的习惯是标注来源。Skill 默认添加来源注释 (HTML 注释，不影响渲染)。
