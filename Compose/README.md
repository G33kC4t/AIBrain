# Compose 源码分析系列文章

深入分析 Jetpack Compose 运行时的核心机制，通过中文技术文章和交互式动画帮助开发者理解 Compose 内部原理。

## 文章列表

### 第一篇：SlotTable 的结构

📄 [阅读文章](01-SlotTable的结构.md)

**内容概览：**
- SlotTable 在 Compose 中的作用与架构定位
- SlotTable 数据结构设计（双数组、Group 结构、Gap Buffer）
- Composable 函数的编译转换过程
- SlotTable 的写入流程与性能优化

---

### 第二篇：SlotTable 与重组

📄 [阅读文章](02-SlotTable与重组.md)

**内容概览：**
- 重组的触发机制
- SlotTable 在重组中的读写流程
- Key 的匹配与复用策略
- 条件渲染与列表的处理

---

### 第三篇：SlotTable 到 LayoutNode

📄 [阅读文章](03-SlotTable到LayoutNode.md)

**内容概览：**
- 从 SlotTable 到 UI 节点树的转换
- LayoutNode 的结构与职责
- ApplyChanges 阶段的工作原理
- 节点树的更新机制

---

### 第四篇：SubcomposeLayout 与复用机制

📄 [阅读文章](04-SubcomposeLayout与复用机制.md)

**内容概览：**
- SubcomposeLayout 的设计动机
- 延迟组合与按需渲染
- LazyColumn 的复用池机制
- 性能优化策略

---

## 交互式动画

🎬 [查看所有动画演示](animations/index.html)

本系列包含 11 个交互式动画：

| 动画 | 说明 |
|------|------|
| group-tree-storage | SlotTable 完整存储可视化 |
| gap-buffer-operations | Gap Buffer 操作演示 |
| group-info-types | Group 类型与 groupInfo 字段 |
| slottable-recomposition | 重组过程可视化 |
| key-matching | Key 匹配机制 |
| key-change-reuse | Key 变化与复用 |
| apply-changes-flow | ApplyChanges 流程 |
| subcompose-lifecycle | SubcomposeLayout 生命周期 |
| subcompose-reuse-pool | 复用池机制 |
| lazy-column-reuse | LazyColumn 复用演示 |

---

## Obsidian 使用说明

### 启用 HTML 动画显示

1. 打开 Obsidian 设置（齿轮图标）
2. 点击左侧菜单的"核心插件"
3. 找到并启用"Web Viewer"插件
4. 在阅读视图（Ctrl/Cmd + E）中查看文章

### 路径约束

⚠️ 本知识库必须保存在以下路径：

```
/Users/tyrionguan/Documents/Obsidian Vault/AIBrain/
```

原因：Markdown 文件中使用绝对路径引用本地 HTML 文件。如果移动位置，需要更新所有 iframe 路径。

---

## 项目结构

```
Compose/
├── README.md                        # 本文件
├── CLAUDE.md                        # AI 助手指南
├── PLAN.md                          # 创作计划
├── 01-SlotTable的结构.md            # 第一篇
├── 02-SlotTable与重组.md            # 第二篇
├── 03-SlotTable到LayoutNode.md      # 第三篇
├── 04-SubcomposeLayout与复用机制.md  # 第四篇
└── animations/                      # 交互式动画
    ├── index.html                   # 动画索引页
    └── *.html                       # 各动画文件
```

---

## 技术栈

- **HTML5 + CSS3**：动画界面
- **JavaScript**：交互逻辑
- **Markdown**：文章内容
- **无依赖**：纯原生实现

---

## 版本信息

- **Compose 版本**：基于 Jetpack Compose 1.7.x
- **最后更新**：2026-02

---

## 参考资源

- [Jetpack Compose 源码](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [Compose 官方文档](https://developer.android.com/jetpack/compose/mental-model)
- [Compose Lifecycle](https://developer.android.com/jetpack/compose/lifecycle)

---

## 许可

本文档仅供学习和研究使用。
