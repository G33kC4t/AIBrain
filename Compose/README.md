# Compose 源码分析系列文章

## 文章列表

### 第一篇：从 Composable 到 SlotTable - 状态槽表的设计原理

📄 [阅读文章](01-compose-slottable-design.md)

**内容概览：**
- SlotTable 在 Compose 中的作用与架构定位
- SlotTable 数据结构设计（双数组、Group 结构、Gap Buffer）
- Composable 函数的编译转换过程
- SlotTable 的写入流程与性能优化

**包含 2 个交互式动画演示：**
1. Counter 示例 - SlotTable 完整存储布局（groups 和 slots 数组的每个值）
2. Gap Buffer 操作可视化

🎬 [查看所有动画演示](animations/index.html)

---

## Obsidian 使用说明

### 启用 HTML 动画显示

本项目的 Markdown 文章中嵌入了交互式 HTML 动画。为了在 Obsidian 中正确显示这些动画，请按以下步骤操作：

#### 一次性配置（必须）

1. 打开 Obsidian 设置（齿轮图标）
2. 点击左侧菜单的"核心插件"
3. 找到"Web Viewer"插件
4. 启用该插件

#### 查看动画

配置完成后，在阅读视图（Ctrl/Cmd + E）中打开任何文章，iframe 动画将自动显示。

### 重要规则

⚠️ **路径约束**：本知识库必须保存在以下路径：

```
/Users/tyrionguan/Documents/Obsidian Vault/AIBrain/
```

原因：Markdown 文件中使用了绝对路径来引用本地 HTML 文件，以确保 Obsidian 能够正确加载动画。如果移动到其他位置，需要更新所有文章中的 iframe 路径。

### 技术细节

所有 iframe 使用 `file://` 协议的绝对路径：

```html
<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/animations/xxx.html"></iframe>
```

这是 Obsidian 能够识别本地 HTML 文件的必要格式。

---

## 如何查看动画

### 方法 1：在浏览器中直接打开

1. 使用浏览器打开 `animations/index.html` 文件
2. 点击任意动画卡片查看完整演示
3. 每个动画都支持交互操作

### 方法 2：在 Markdown 阅读器中查看

如果你的 Markdown 阅读器支持嵌入 HTML（如 Typora、VS Code、GitHub），可以直接在文章中看到嵌入的动画。

### 方法 3：启动本地服务器

如果嵌入的 iframe 无法正常显示，可以启动一个本地服务器：

```bash
# 使用 Python
python3 -m http.server 8000

# 或使用 Node.js
npx http-server

# 然后在浏览器访问
# http://localhost:8000/01-compose-slottable-design.md
# http://localhost:8000/animations/index.html
```

---

## 动画说明

### 1. Counter 示例 - SlotTable 完整存储 (group-tree-storage.html)

**示例代码：**
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

**展示内容：**
- 左侧：6 个 Group 的完整树形结构
  - Group 0: Counter (Restartable)
  - Group 1: remember
  - Group 2: Column (Node)
  - Group 3: Text
  - Group 4: Button (Node)
  - Group 5: Text
- 右侧：groups 数组的完整存储（30 个 Int 值）
  - 每个 Group 占用 5 个 Int
  - 详细展示 key、groupInfo、parentAnchor、size、dataAnchor
- 右侧：slots 数组的完整内容（7 个槽位）
  - [0]: MutableState(0) - remember 的结果
  - [1]: LayoutNode - Column 的节点对象
  - [2]: Modifier.Companion
  - [3]: "Count: 0" - Text 的文本
  - [4]: LayoutNode - Button 的节点对象
  - [5]: lambda { count++ } - Button 的 onClick
  - [6]: "Increment" - 按钮内 Text 的文本

**特点：**
- 双视图对比：树形结构 vs 数组存储
- 详细标注每个字段的含义
- 动画展示加载过程
- 包含 remember 的实际应用示例

### 2. Gap Buffer 操作演示 (gap-buffer-operations.html)

**展示内容：**
- 初始状态：gap 的位置和长度
- 插入操作：在 gap 中写入数据
- 删除操作：扩大 gap
- 移动 Gap：数据复制过程

**特点：**
- 交互式操作选择
- 分步演示
- 实时状态显示
- 支持前进/后退/重置

---

## 技术栈

- **HTML5 + CSS3**：动画界面
- **JavaScript**：交互逻辑
- **Markdown**：文章内容
- **无依赖**：纯原生实现，无需安装任何框架

---

## 项目结构

```
AIBrain/
├── 01-compose-slottable-design.md          # 第一篇文章
├── animations/                               # 动画文件夹
│   ├── index.html                           # 动画索引页
│   ├── group-tree-storage.html              # 动画1：SlotTable 完整存储
│   └── gap-buffer-operations.html           # 动画2：Gap Buffer 操作
├── test-animation.html                      # 动画测试页面
└── README.md                                # 本文件
```

---

## 下一步计划

- [ ] 第二篇：SlotTable 到节点树 - UI 树的构建与更新机制
- [ ] 第三篇：SubCompose 深度解析 - 动态组合与性能优化

---

## 版本信息

- **Compose 版本**：基于 Jetpack Compose 1.7.x 最新稳定版
- **文章创建日期**：2026-01-30
- **作者**：AI 辅助创作

---

## 参考资源

- [Jetpack Compose 源码](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [Compose 官方文档](https://developer.android.com/jetpack/compose/mental-model)
- [Understanding Composition](https://developer.android.com/jetpack/compose/lifecycle)

---

## 许可

本文档仅供学习和研究使用。
