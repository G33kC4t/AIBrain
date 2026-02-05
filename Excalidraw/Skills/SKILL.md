---
name: obsidian-excalidraw
description: Create and edit Excalidraw diagrams in Obsidian. Use when creating flowcharts, architecture diagrams, mind maps, or visual drawings in Obsidian's Excalidraw plugin format (.excalidraw.md files).
---

# Obsidian Excalidraw Skill

在 Obsidian 中使用 Excalidraw 创建专业图表的完整指南。

## 文件格式

Obsidian Excalidraw 使用 `.excalidraw.md` 文件格式，结合 Markdown 和 JSON：

```markdown
---
excalidraw-plugin: parsed
tags:
  - excalidraw
---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠==

# Excalidraw Data

## Text Elements
文本内容 ^元素ID

%%
## Drawing
```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://github.com/zsviczian/obsidian-excalidraw-plugin/...",
  "elements": [...],
  "appState": {...},
  "files": {}
}
` ` `
%%
```

### 关键格式规则

| 部分 | 说明 |
|------|------|
| Frontmatter | `excalidraw-plugin: parsed` 必需 |
| Text Elements | 文本内容提取，格式：`文本 ^ID` |
| Drawing | JSON 在 `%%` 注释块内 |

## 元素类型速查

### 基础图形

| 类型 | 用途 | 必需属性 |
|------|------|----------|
| `rectangle` | 矩形、容器、卡片 | x, y, width, height |
| `ellipse` | 圆形、椭圆、节点 | x, y, width, height |
| `diamond` | 菱形、决策点 | x, y, width, height |
| `text` | 文本标签 | x, y, text, fontSize |
| `arrow` | 箭头连接 | x, y, points |
| `line` | 线条连接 | x, y, points |
| `frame` | 分组框架 | x, y, name, children |

### 最小元素模板

**矩形**：
```json
{
  "type": "rectangle",
  "id": "rect-1",
  "x": 100, "y": 100,
  "width": 200, "height": 100,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#a5d8ff",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "angle": 0,
  "seed": 12345,
  "version": 1,
  "versionNonce": 12345,
  "isDeleted": false,
  "groupIds": [],
  "boundElements": null,
  "link": null,
  "locked": false,
  "roundness": { "type": 3 }
}
```

**文本**：
```json
{
  "type": "text",
  "id": "text-1",
  "x": 120, "y": 130,
  "width": 160, "height": 25,
  "text": "标签文本",
  "fontSize": 20,
  "fontFamily": 2,
  "textAlign": "center",
  "verticalAlign": "middle",
  "containerId": null,
  "originalText": "标签文本",
  "autoResize": true,
  "lineHeight": 1.25,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "angle": 0,
  "seed": 12346,
  "version": 1,
  "versionNonce": 12346,
  "isDeleted": false,
  "groupIds": [],
  "boundElements": null,
  "link": null,
  "locked": false
}
```

**箭头**：
```json
{
  "type": "arrow",
  "id": "arrow-1",
  "x": 300, "y": 150,
  "width": 100, "height": 0,
  "points": [[0, 0], [100, 0]],
  "startArrowhead": null,
  "endArrowhead": "arrow",
  "startBinding": null,
  "endBinding": null,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "angle": 0,
  "seed": 12347,
  "version": 1,
  "versionNonce": 12347,
  "isDeleted": false,
  "groupIds": [],
  "boundElements": null,
  "link": null,
  "locked": false
}
```

## 样式参考

### 颜色调色板

**背景色**：
| 颜色 | 十六进制 | 语义 |
|------|----------|------|
| 浅蓝 | `#a5d8ff` | 信息、输入、用户 |
| 浅绿 | `#b2f2bb` | 成功、输出、数据库 |
| 浅黄 | `#ffec99` | 警告、决策、流程 |
| 浅红 | `#ffc9c9` | 错误、危险、停止 |
| 浅紫 | `#d0bfff` | 外部、存储、特殊 |
| 浅灰 | `#e9ecef` | 中性、禁用、备注 |
| 浅青 | `#99e9f2` | 负载均衡、网络 |

**描边色**：
| 颜色 | 十六进制 |
|------|----------|
| 默认黑 | `#1e1e1e` |
| 蓝色 | `#1971c2` |
| 绿色 | `#2f9e44` |
| 橙色 | `#f08c00` |
| 红色 | `#e03131` |
| 紫色 | `#9c36b5` |
| 灰色 | `#868e96` |

### 样式属性

| 属性 | 值 | 说明 |
|------|-----|------|
| `fillStyle` | `"solid"`, `"hachure"`, `"cross-hatch"` | 填充样式 |
| `strokeWidth` | `1`, `2`, `4` | 线条粗细 |
| `strokeStyle` | `"solid"`, `"dashed"`, `"dotted"` | 线条样式 |
| `roughness` | `0`, `1`, `2` | 手绘感（0=精确，2=粗糙） |
| `opacity` | `0-100` | 透明度 |
| `roundness` | `{ "type": 3 }` | 圆角 |

### 字体

| fontFamily | 字体 | 用途 |
|------------|------|------|
| `1` | Virgil（手写） | 休闲、草图 |
| `2` | Helvetica | 专业、正式（推荐） |
| `3` | Cascadia（等宽） | 代码、技术 |

### 字号指南

| 用途 | 字号 |
|------|------|
| 主标题 | 28-36 |
| 节标题 | 24 |
| 框内标签 | 20 |
| 描述文字 | 16 |
| 注释 | 14 |

## 元素绑定

### 文本绑定到容器

容器元素：
```json
{
  "type": "rectangle",
  "id": "rect-1",
  "boundElements": [{ "id": "text-1", "type": "text" }]
}
```

文本元素：
```json
{
  "type": "text",
  "id": "text-1",
  "containerId": "rect-1",
  "textAlign": "center",
  "verticalAlign": "middle"
}
```

### 箭头绑定到图形

箭头元素：
```json
{
  "type": "arrow",
  "id": "arrow-1",
  "startBinding": {
    "elementId": "rect-1",
    "focus": 0,
    "gap": 5
  },
  "endBinding": {
    "elementId": "rect-2",
    "focus": 0,
    "gap": 5
  }
}
```

图形元素：
```json
{
  "type": "rectangle",
  "id": "rect-1",
  "boundElements": [{ "id": "arrow-1", "type": "arrow" }]
}
```

## 图表模式

### 流程图

**标准符号**：
| 图形 | 含义 | 颜色 |
|------|------|------|
| 椭圆 | 开始/结束 | `#b2f2bb` 绿 |
| 矩形 | 处理步骤 | `#a5d8ff` 蓝 |
| 菱形 | 决策分支 | `#ffec99` 黄 |
| 圆形 | 连接点 | `#e9ecef` 灰 |

**布局规则**：
- 方向：上→下 或 左→右
- 元素间距：100px
- 决策箭头标注 Yes/No

### 架构图

**组件颜色**：
| 组件 | 图形 | 颜色 |
|------|------|------|
| 客户端/用户 | 矩形/椭圆 | `#a5d8ff` 蓝 |
| 服务/API | 矩形 | `#b2f2bb` 绿 |
| 数据库 | 椭圆 | `#d0bfff` 紫 |
| 消息队列 | 矩形 | `#ffec99` 黄 |
| 外部系统 | 矩形 | `#e9ecef` 灰 |

**连接标注**：
- 协议：HTTP, gRPC, SQL
- 箭头样式：实线=同步，虚线=异步

### 时序图

**组件**：
- 参与者：顶部矩形
- 生命线：垂直虚线（`strokeStyle: "dashed"`）
- 消息：水平箭头
- 返回：虚线箭头

**布局**：
- 参与者间距：200px
- 消息垂直间距：50-80px
- 消息编号：1., 2., 3.

### 思维导图

**层级样式**：
| 层级 | 字号 | 尺寸 | 线宽 |
|------|------|------|------|
| 中心 | 28-36 | 180×100 | 4 |
| 一级 | 20-24 | 140×70 | 2 |
| 二级 | 16-18 | 100×50 | 2 |
| 三级 | 14-16 | 80×40 | 1 |

## Obsidian 特有功能

### 链接语法

在文本元素中使用 Obsidian 链接：
```
[[页面名称]]
[[页面名称|显示文本]]
[[页面名称#标题]]
```

### 嵌入内容

嵌入其他笔记：
```
![[笔记名称]]
```

嵌入图片：
```
![[图片.png]]
```

### 脚本引擎

常用脚本功能：
| 脚本 | 功能 |
|------|------|
| Fixed spacing | 固定间距排列元素 |
| Expand rectangles | 统一矩形尺寸 |
| Connect elements | 箭头连接对象 |
| Text Align | 文本对齐 |
| Font Family | 字体切换 |
| Darken/Lighten color | 颜色亮度调节 |

## 最佳实践

### 创建图表步骤

1. **规划布局** - 先在纸上草拟
2. **创建图形** - 先画所有形状
3. **添加文本** - 绑定文本到容器
4. **连接箭头** - 最后添加箭头并绑定
5. **调整样式** - 统一颜色和字体

### 设计原则

- **一致性** - 相同概念用相同图形
- **间距** - 使用 50px 或 100px 的倍数
- **颜色** - 最多 2-3 种强调色
- **字体** - 正式文档用 Helvetica
- **粗糙度** - 技术文档用 `roughness: 0`

### 常见错误

| 错误 | 正确做法 |
|------|----------|
| 颜色过多 | 限制 2-3 种 |
| 间距不一 | 统一间距 |
| 文字太小 | 最小 14px |
| 箭头交叉 | 重新路由 |
| 未绑定元素 | 使用 boundElements |

## 快速模板

### 专业风格

```json
{
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#e9ecef",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 0,
  "opacity": 100,
  "fontFamily": 2,
  "fontSize": 16
}
```

### 高亮元素

```json
{
  "backgroundColor": "#a5d8ff",
  "strokeColor": "#1971c2"
}
```

## 相关文档

- [element-reference.md](element-reference.md) - 元素属性完整参考
- [diagram-patterns.md](diagram-patterns.md) - 图表模式详解
- [best-practices.md](best-practices.md) - 设计最佳实践
- [examples.md](examples.md) - 完整示例
