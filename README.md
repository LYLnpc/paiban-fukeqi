# 排版复刻器

把网页设计稿自动导出为 AE 可重建的图层 JSON，一键重建合成。

## 这是什么？

排版复刻器 是一套「网页 → AE」的图层迁移工具。

你给 AI 一张排版图，它帮你生成一份 HTML；你在浏览器里点几下导出 JSON；回到 AE 运行脚本，一个基础排版就自动生成好了。

不用手动量坐标，不用在 AE 里重新打字，不用逐个画矩形。

## 使用说明

1. 找到一个拥有视觉模型的 AI
   Claude、GPT、Gemini 等支持读图的模型均可。

2. 把 skill 复制给它
   打开仓库里的 SKILL.md，全选复制，粘贴进对话。

3. 把排版图一起输入
   把你的海报 / 封面 / 卡片 / UI 截图拖进同一个对话。

4. 稍微等待
   AI 会读图 → 拆解视觉结构 → 生成一份完整单页 HTML。

5. 点击运行 HTML
   把 AI 输出的 HTML 保存为 .html 文件，用 Chrome / Edge 打开。
   左上角应显示 EXPORTER v6 — READY。

6. 点击扫描
   右上角控制面板 → 点【扫描】，确认图层统计正常。

7. 点击下载
   点【下载】，得到 layout.json。

8. 回到 AE 打开脚本
   运行 排版复刻器.jsx（放到 Scripts/ScriptUI Panels/ 或直接 File > Scripts > Run Script File）。

9. 导入 JSON
   在脚本面板里选择刚才下载的 layout.json。

10. 点击生成
    脚本自动新建合成，按 JSON 正序创建所有图层。一个基础排版就生成好了。

## 功能亮点

- **文字图层名 = 文字本体**：时间轴上一眼可读，不用再靠 text-1 / shape-2 猜。
- **背景与文字顺序正确**：同元素 bg + text 按 CSS 绘制顺序导出，AE 里文字不会被自己的背景盖住。
- **SVG 图形全支持**：line / path / circle / rect / polyline / polygon / text 均可导出为可编辑折线。
- **画幅预设 + 自定义尺寸**：内置 9:16 / 3:4 / 4:5 / 1:1 / 16:9，也可手动输入宽高实时预览。
- **控制面板可折叠**：不挡画布，小屏也能用。
- **超高画布不裁顶**：960×2112 也能完整滚动查看。
- **AE 端脚本开箱即用**：读 JSON 正序建层，叠放关系与 DOM 完全一致。

## 项目组成

| 文件 | 说明 |
| --- | --- |
| SKILL.md | 给 AI 的完整 skill 文档（复制给 AI 用） |
| ae-layout-exporter.v6.html | 浏览器端提取器，导出图层 JSON |
| 排版复刻器.jsx | AE 端脚本，读取 layout.json 重建合成 |
| layout.json | 导出结果示例 |

## 图层命名规则

文字图层的 name 直接等于它要显示的文字本体。命名优先级：

1. `data-ae-name` / `data-name` / `name` / `aria-label` / `title` 显式覆盖
2. 文字本体（如 `春日限定 · 樱花季 🌸`）
3. 元素 class（形状 / 无文字元素，如 `card-panel bg`）
4. 兜底占位（`text` / `shape` / `svg-text-N`）

同名自动加序号：`限时特惠` / `限时特惠_2` / `限时特惠_3`。

清洗规则：换行折成空格；删除 `\ / * ? " < > | [ ]`；冒号保留（时间文本 `0:00` 合法）；超长按词边界截断加 `…`（默认 64 字符）。

## SVG 支持

扫描时遇到 `<svg>` 元素，走独立通道：

| SVG 标签 | 输出类型 | 说明 |
| --- | --- | --- |
| `<line>` | polygon | 两个端点（专用通道，最可靠） |
| `<path>` | polygon | 按 getPointAtLength 采样，密度 min(600, max(24, len/4)) |
| `<circle>` / `<ellipse>` | polygon | 36 段折线采样 |
| `<rect>` | polygon | 4 个角点 |
| `<polyline>` / `<polygon>` | polygon | 顶点列表 |
| `<text>` | text | 携带字体、字号、颜色 |

不支持：`<use>` / `<symbol>` / `<defs>` / clipPath / mask。

## 输出 JSON 结构

```jsonc
{
  "width": 960,
  "height": 2112,
  "layers": [
    {
      "type": "text",
      "name": "Hello World",
      "x": 50, "y": 200, "w": 200, "h": 40,
      "text": "Hello World",
      "font": "Arial",
      "fontSize": 32,
      "fontWeight": 400,
      "color": {"r":0.2,"g":0.2,"b":0.2,"a":1},
      "align": "left",
      "lineHeight": 38.4,
      "tracking": 0,
      "opacity": 1,
      "rotate": 0
    }
    // ... rect / gradient / polygon
  ]
}
```

坐标单位：像素，原点为容器左上角。颜色：0~1 的 RGBA。

## AE 端注意事项

- **叠放顺序**：脚本按 JSON 数组正序创建图层，最终从下到上 = 数组顺序 = DOM 顺序。
- **段落文本框**：锚点在框中心，Position 需设 `[x + w/2, y + h/2]`。
- **tracking 取整**：AE 的 tracking 只接受整数，脚本需 `Math.round`。
- **字体映射**：CSS 字体名需映射为 PostScript 名；通用族 `serif` → `TimesNewRomanPSMT`，`sans-serif` → `ArialMT`。
- **渐变降级**：AE 脚本无法写入渐变色标，只能取首色标降级为纯色；写色失败需 `gf.remove()`。
- **增补平面字符**：如 `𝄞` 做文本层会变空层，装饰图形一律走 SVG。

## 最小验证用例

替换 `.container` 内部为以下内容，刷新后点【扫描】：

```html
<div style="position:absolute;left:50px;top:50px;width:200px;height:100px;background:#3a7bd5;border-radius:16px;"></div>
<div style="position:absolute;left:50px;top:200px;color:#333;font-size:32px;">Hello World</div>
<svg style="position:absolute;left:50px;top:300px;width:400px;height:200px;">
  <line x1="0" y1="0" x2="400" y2="200" stroke="#f00" stroke-width="2" stroke-dasharray="4 8"/>
  <circle cx="100" cy="100" r="60" fill="#0f0" stroke="#333" stroke-width="2"/>
  <path d="M 0 0 Q 100 -80 200 0 T 400 0" fill="none" stroke="#00f" stroke-width="3"/>
</svg>
<svg style="position:absolute;left:80px;top:560px;width:60px;height:80px;">
  <ellipse cx="18" cy="62" rx="11" ry="8" fill="#333" transform="rotate(-22 18 62)"/>
  <rect x="27" y="10" width="3" height="52" fill="#333"/>
  <path d="M27 10 Q 48 18 27 48 L 27 30 Q 42 24 28 12 Z" fill="#333"/>
</svg>
```

预期扫描统计（左上角徽章应为 EXPORTER v6 — READY）：

```
✅ 共 8 层
文本: 1
矩形/渐变: 2
SVG线条: 1
其他形状: 4
```

数字对不上 → 大概率缓存了旧版，检查版本徽章。

## 常见问题

**Q：为什么扫描结果里没有我的图片？**
A：background-image: url(...) 和 `<img>` 不解析，图片需在 AE 手动导入。

**Q：为什么文字图层名和我想的不一样？**
A：默认取文字本体。想自定义，加 `data-ae-name="想要的名称"`。

**Q：为什么 AE 里文字被背景盖住了？**
A：确认导出顺序：背景层应在文字层之前（数组索引更小）。v6 已修复，若仍出现请检查是否用了旧版 HTML。

**Q：为什么渐变在 AE 里变成纯色？**
A：AE 脚本无法写渐变色标，只能取首色标降级。需手动在 AE 里调渐变。

**Q：图标导出后是空白或方块？**
A：用了 Unicode 字符。请改用 `<svg>` 绘制图标。

## 贡献

欢迎提 Issue 或 PR。提交前请确保：

- 提取器代码不破坏现有命名规则与绘制顺序；
- 最小验证用例扫描统计与预期一致；
- 若修改 AE 脚本，注明 AE 版本与实机验证结果。

## 许可证

MIT © 2026 LYLnpc

## 致谢

灵感来自 AE 手动重建网页稿的重复劳动。

感谢所有在 AE 实机测试中踩过的坑。

---

赞助主播继续创作：https://github.com/LYLnpc/Tipping-channel
