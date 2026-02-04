# CLAUDE.md - Compose 子项目指南

> 此文件为 Compose 源码分析系列文章子项目的工作指导

## 项目概述

本子项目专注于分析 Jetpack Compose 运行时的核心机制，通过中文技术文章和交互式动画帮助开发者深入理解 Compose 内部原理。

**项目路径**：`/Users/tyrionguan/Documents/Obsidian Vault/AIBrain/Compose/`

## 目录结构

```
Compose/
├── CLAUDE.md                           # 本文件：子项目工作指南
├── PLAN.md                             # 创作计划和进度跟踪
├── README.md                           # 项目介绍文档
├── 01-SlotTable的结构.md               # 第一篇
├── 02-SlotTable与重组.md               # 第二篇
├── 03-SlotTable到LayoutNode.md         # 第三篇
├── 04-SubcomposeLayout与复用机制.md     # 第四篇
└── animations/                         # 交互式动画目录
    ├── index.html                      # 动画索引页
    ├── group-tree-storage.html         # SlotTable 存储可视化
    ├── gap-buffer-operations.html      # Gap Buffer 操作演示
    ├── group-info-types.html           # Group 类型与 groupInfo 字段
    ├── slottable-recomposition.html    # 重组过程可视化
    ├── key-matching.html               # Key 匹配机制
    ├── key-change-reuse.html           # Key 变化与复用
    ├── apply-changes-flow.html         # ApplyChanges 流程
    ├── subcompose-lifecycle.html       # SubcomposeLayout 生命周期
    ├── subcompose-reuse-pool.html      # 复用池机制
    └── lazy-column-reuse.html          # LazyColumn 复用演示
```

## 文件命名规范

- **文章文件**：`[序号]-[中文主题].md`
  - 例如：`01-SlotTable的结构.md`
  - 例如：`02-SlotTable与重组.md`
- **动画文件**：`[描述性名称].html`（小写字母 + 连字符）

## 系列文章

| 序号 | 文件名 | 主题 | 状态 |
|------|--------|------|------|
| 01 | `01-SlotTable的结构.md` | SlotTable 数据结构设计 | ✅ 已完成 |
| 02 | `02-SlotTable与重组.md` | 重组机制与 Key 匹配 | ✅ 已完成 |
| 03 | `03-SlotTable到LayoutNode.md` | UI 节点树构建 | ✅ 已完成 |
| 04 | `04-SubcomposeLayout与复用机制.md` | 延迟组合与复用池 | ✅ 已完成 |

详细计划请参阅 `PLAN.md`。

## iframe 路径规则

**重要**：在 Markdown 文章中嵌入动画时，必须使用绝对路径：

```html
<!-- ✅ 正确：Obsidian 专用绝对路径 -->
<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/gap-buffer-operations.html" ...></iframe>

<!-- ❌ 错误：相对路径在 Obsidian 中无法加载 -->
<iframe src="animations/gap-buffer-operations.html" ...></iframe>
```

**路径模板**：
```
file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/[文件名].html
```

## 动画开发规范

### 技术要求
- 纯 HTML/CSS/JavaScript，无外部依赖
- 自包含单文件，可独立在浏览器中运行
- 响应式设计，适配 iframe 嵌入

### 视觉风格
- 主色调：`#4a90d9`（蓝色）
- 背景色：`#f8f9fa`（浅灰）
- 代码背景：`#1e1e1e`（深色）
- 圆角：`8px`
- 字体：系统字体栈

### 交互要求
- 必须有播放/暂停/重置控制
- 支持分步动画和手动控制
- 清晰的中文标签和说明

## 写作风格

### 文章结构
1. **引言**：背景、动机、内容概述
2. **分步解析**：源码 → 编译转换 → 运行时数据结构
3. **深入章节**：算法细节、性能优化、边界情况
4. **总结**：关键要点、与下一篇的联系

### 代码示例
- 使用真实的 Kotlin/Compose 代码
- 关键代码添加注释
- 编译后伪代码保持准确但简化

### 技术术语
- 保留英文原文：SlotTable、Group、Gap Buffer、Anchor 等
- 首次出现时附中文解释
- 全文术语保持一致

## 工作流程

### 创建新文章
1. 在 `PLAN.md` 中确认文章大纲
2. 创建文章文件 `[序号]-[主题].md`
3. 创建配套动画文件
4. 更新 `animations/index.html` 添加新动画卡片
5. 更新 `PLAN.md` 标记完成状态

### 修订现有文章
- 小修改：直接编辑
- 重大修改：先创建备份

### 提交规范
- 提交信息使用中文
- 每次提交应是一个逻辑完整的改动

## 参考资源

### Compose 源码
- [SlotTable.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SlotTable.kt)
- [Composer.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composer.kt)
- [Composition.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composition.kt)

### 官方文档
- [Compose Mental Model](https://developer.android.com/jetpack/compose/mental-model)
- [Compose Lifecycle](https://developer.android.com/jetpack/compose/lifecycle)

## AI 助手须知

### 语言要求
- 所有对话使用**简体中文**
- 文章内容使用简体中文
- 代码注释可中英文混合

### 内容创作原则
- 基于实际 Compose 源码，确保技术准确
- 代码示例使用真实的 Kotlin 代码
- 动画自包含，无外部依赖
- 与现有文章风格保持一致

### 关键提醒
1. **路径问题**：iframe 必须使用绝对路径
2. **进度跟踪**：及时更新 `PLAN.md`
3. **术语一致**：使用统一的技术术语

---

*此文件为 Compose 子项目的专属指南，通用规则请参阅根目录的 `CLAUDE.md`*
