# Excalidraw 元素参考

所有 Excalidraw 元素类型的完整属性参考。

## 通用属性

每个元素必须包含以下属性：

```json
{
  "id": "unique-id",
  "type": "rectangle",
  "x": 0,
  "y": 0,
  "width": 100,
  "height": 100,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "seed": 12345,
  "version": 1,
  "versionNonce": 12345,
  "isDeleted": false,
  "groupIds": [],
  "boundElements": null,
  "link": null,
  "locked": false
}
```

### 属性详解

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识符，建议使用描述性名称如 `"header-box"` |
| `type` | string | 元素类型 |
| `x` | number | X 坐标（左边缘） |
| `y` | number | Y 坐标（上边缘） |
| `width` | number | 宽度（像素） |
| `height` | number | 高度（像素） |
| `angle` | number | 旋转角度（弧度，0 = 无旋转） |
| `strokeColor` | string | 边框/线条颜色（十六进制） |
| `backgroundColor` | string | 填充颜色（十六进制或 `"transparent"`） |
| `fillStyle` | string | 填充样式 |
| `strokeWidth` | number | 线条粗细 |
| `strokeStyle` | string | 线条样式 |
| `roughness` | number | 手绘粗糙度 |
| `opacity` | number | 透明度（0-100） |
| `seed` | number | 随机种子（手绘渲染一致性） |
| `version` | number | 元素版本（初始为 1） |
| `versionNonce` | number | 版本随机数 |
| `isDeleted` | boolean | 软删除标记（使用 `false`） |
| `groupIds` | array | 所属分组 ID 数组 |
| `boundElements` | array/null | 绑定的元素（箭头、文本） |
| `link` | string/null | 附加的 URL 链接 |
| `locked` | boolean | 锁定编辑 |

---

## 矩形 Rectangle

```json
{
  "type": "rectangle",
  "id": "rect-1",
  "x": 100,
  "y": 100,
  "width": 200,
  "height": 100,
  "roundness": { "type": 3 },
  // ... 通用属性
}
```

### 矩形特有属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `roundness` | object/null | `{ "type": 3 }` 圆角，`null` 直角 |

---

## 椭圆 Ellipse

```json
{
  "type": "ellipse",
  "id": "ellipse-1",
  "x": 100,
  "y": 100,
  "width": 150,
  "height": 150,
  // ... 通用属性
}
```

**注意**：`width` 等于 `height` 时为正圆。

---

## 菱形 Diamond

```json
{
  "type": "diamond",
  "id": "diamond-1",
  "x": 100,
  "y": 100,
  "width": 120,
  "height": 120,
  // ... 通用属性
}
```

菱形是旋转的正方形，常用于流程图决策节点。

---

## 文本 Text

```json
{
  "type": "text",
  "id": "text-1",
  "x": 100,
  "y": 100,
  "width": 200,
  "height": 50,
  "text": "文本内容",
  "fontSize": 20,
  "fontFamily": 5,
  "textAlign": "left",
  "verticalAlign": "top",
  "containerId": null,
  "originalText": "文本内容",
  "autoResize": true,
  "lineHeight": 1.25,
  // ... 通用属性
}
```

### 文本特有属性

| 属性 | 类型 | 值 |
|------|------|-----|
| `text` | string | 显示文本（支持 `\n` 换行） |
| `fontSize` | number | 常用：`16`, `20`, `28`, `36` |
| `fontFamily` | number | `5`（Excalidraw 默认，**推荐**）, `1`（Virgil）, `2`（Helvetica）, `3`（Cascadia） |
| `textAlign` | string | `"left"`, `"center"`, `"right"` |
| `verticalAlign` | string | `"top"`, `"middle"` |
| `containerId` | string/null | 容器元素 ID（绑定时使用） |
| `originalText` | string | 原始文本（与 `text` 相同） |
| `autoResize` | boolean | 自动调整容器大小 |
| `lineHeight` | number | 行高倍数（默认 1.25） |

---

## 箭头 Arrow

```json
{
  "type": "arrow",
  "id": "arrow-1",
  "x": 100,
  "y": 100,
  "width": 200,
  "height": 0,
  "points": [[0, 0], [200, 0]],
  "startArrowhead": null,
  "endArrowhead": "arrow",
  "startBinding": null,
  "endBinding": null,
  // ... 通用属性
}
```

### 箭头特有属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `points` | array | 相对于元素原点的 [x, y] 点数组 |
| `startArrowhead` | string/null | `null`, `"arrow"`, `"bar"`, `"dot"`, `"triangle"` |
| `endArrowhead` | string/null | 同上 |
| `startBinding` | object/null | 起点绑定 |
| `endBinding` | object/null | 终点绑定 |

### 箭头路径示例

```json
// 水平直线（200px 长）
"points": [[0, 0], [200, 0]]

// 垂直直线（150px 向下）
"points": [[0, 0], [0, 150]]

// L 形
"points": [[0, 0], [100, 0], [100, 100]]

// 曲线（3+ 点自动平滑）
"points": [[0, 0], [50, -30], [100, 0]]
```

### 绑定对象

```json
{
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

| 属性 | 类型 | 说明 |
|------|------|------|
| `elementId` | string | 绑定元素的 ID |
| `focus` | number | -1 到 1，边缘位置（0 = 中心） |
| `gap` | number | 箭头与图形之间的间距（像素） |

---

## 线条 Line

与箭头相同，但默认无箭头：

```json
{
  "type": "line",
  "id": "line-1",
  "x": 100,
  "y": 100,
  "width": 200,
  "height": 0,
  "points": [[0, 0], [200, 0]],
  "startArrowhead": null,
  "endArrowhead": null,
  // ... 通用属性
}
```

---

## 自由绘制 Freedraw

手绘路径：

```json
{
  "type": "freedraw",
  "id": "freedraw-1",
  "x": 100,
  "y": 100,
  "width": 150,
  "height": 80,
  "points": [[0, 0], [10, 5], [20, 3], [30, 8], ...],
  "pressures": [0.5, 0.6, 0.7, ...],
  "simulatePressure": true,
  // ... 通用属性
}
```

### Freedraw 特有属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `points` | array | 密集的 [x, y] 点数组 |
| `pressures` | array | 每个点的压力值 0-1 |
| `simulatePressure` | boolean | 模拟笔压 |

---

## 框架 Frame

视觉分组：

```json
{
  "type": "frame",
  "id": "frame-1",
  "x": 50,
  "y": 50,
  "width": 500,
  "height": 400,
  "name": "框架名称",
  // ... 通用属性
}
```

子元素应设置 `frameId` 为框架的 ID。

---

## 图片 Image

```json
{
  "type": "image",
  "id": "image-1",
  "x": 100,
  "y": 100,
  "width": 300,
  "height": 200,
  "fileId": "file-abc123",
  "status": "saved",
  "scale": [1, 1],
  // ... 通用属性
}
```

图片数据存储在顶层 `files` 对象中：

```json
{
  "files": {
    "file-abc123": {
      "id": "file-abc123",
      "mimeType": "image/png",
      "dataURL": "data:image/png;base64,..."
    }
  }
}
```

---

## 元素绑定

### 图形绑定文本

图形元素：
```json
{
  "type": "rectangle",
  "id": "rect-1",
  "boundElements": [
    { "id": "text-1", "type": "text" }
  ]
}
```

文本元素：
```json
{
  "type": "text",
  "id": "text-1",
  "containerId": "rect-1"
}
```

### 图形绑定箭头

```json
{
  "type": "rectangle",
  "id": "rect-1",
  "boundElements": [
    { "id": "arrow-1", "type": "arrow" }
  ]
}
```

---

## 分组

相同 `groupIds` 的元素属于同一组：

```json
{
  "type": "rectangle",
  "id": "rect-1",
  "groupIds": ["group-1"]
}
{
  "type": "text",
  "id": "text-1",
  "groupIds": ["group-1"]
}
```

嵌套分组使用多个 ID（最内层在前）：
```json
"groupIds": ["inner-group", "outer-group"]
```

---

## 样式值参考

### fillStyle

| 值 | 效果 |
|-----|------|
| `"solid"` | 纯色填充 |
| `"hachure"` | 斜线填充 |
| `"cross-hatch"` | 交叉线填充 |

### strokeStyle

| 值 | 效果 |
|-----|------|
| `"solid"` | 实线 |
| `"dashed"` | 虚线 |
| `"dotted"` | 点线 |

### roughness

| 值 | 效果 | 适用场景 |
|-----|------|----------|
| `1` | 轻微手绘 | **默认推荐**，保持 Excalidraw 特色 |
| `0` | 精确线条 | 正式文档、技术规格 |
| `2` | 明显草图 | 头脑风暴、非正式 |

### strokeWidth

| 值 | 粗细 |
|-----|------|
| `1` | 细线 |
| `2` | 中等（默认） |
| `4` | 粗线 |

### fontFamily

| 值 | 字体 | 用途 |
|-----|------|------|
| `5` | Excalidraw 默认 | **推荐**，保持手绘风格 |
| `1` | Virgil（手写） | 休闲、草图 |
| `2` | Helvetica（无衬线） | 专业、正式 |
| `3` | Cascadia（等宽） | 代码、技术 |

> **推荐**：使用 `fontFamily: 5` 配合 `roughness: 1`，保持 Excalidraw 的手绘特色。

### arrowhead

| 值 | 效果 |
|-----|------|
| `null` | 无箭头 |
| `"arrow"` | 标准箭头 |
| `"bar"` | 竖线 |
| `"dot"` | 圆点 |
| `"triangle"` | 三角形 |
