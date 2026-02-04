# CLAUDE.md

本文件为 Claude Code 在此知识库中工作时提供通用指导。

## 知识库概述

这是一个 **Obsidian 知识库**（AIBrain），用于存储和组织各类技术学习内容。知识库采用子项目结构，每个主题独立管理。

**知识库路径**：`/Users/tyrionguan/Documents/Obsidian Vault/AIBrain/`
**内容语言**：简体中文
**编辑环境**：Obsidian

## 目录结构

```
AIBrain/
├── CLAUDE.md                           # 本文件：通用 AI 助手指南
├── .gitignore                          # Git 忽略规则
│
└── Compose/                            # 子项目：Compose 源码分析系列
    ├── CLAUDE.md                       # 子项目专属指南
    ├── PLAN.md                         # 创作计划和进度
    ├── README.md                       # 项目介绍
    ├── 01-SlotTable的结构.md           # 第一篇
    ├── 02-SlotTable与重组.md           # 第二篇
    ├── 03-SlotTable到LayoutNode.md     # 第三篇
    ├── 04-SubcomposeLayout与复用机制.md # 第四篇
    └── animations/                     # 交互式动画（11个）
```

## 子项目列表

| 目录 | 主题 | 状态 |
|------|------|------|
| `Compose/` | Jetpack Compose 源码分析 | 🟢 进行中（4篇已完成） |

## 通用规范

### 语言要求
- **所有对话使用简体中文**
- 技术术语保留英文原文，必要时附中文解释
- 代码注释可中英文混合

### 项目性质
- 这是一个**文档项目**，不是软件应用程序
- 没有构建命令、测试或部署流程
- 专注于内容准确性、清晰度和教育价值

### 内容创作原则
- **准确性**：基于实际源码，不猜测或编造
- **代码示例**：使用真实代码片段
- **可视化**：复杂概念配合交互式动画
- **渐进式**：从高层概念到实现细节

### 非文字模块标题规则

**所有非文字模块必须添加标题**，方便后续修改时快速定位：

| 模块类型 | 标题格式示例 |
|---------|-------------|
| 表格 | `**slotId 的作用**：` 或 `### 性能对比表` |
| 流程图/示意图 | `**标准组合的限制**：` 或 `### 复用决策流程` |
| iframe 动画 | `> 💡 **交互动画**：下方动画演示了...` |
| 代码块 | 添加注释说明用途，如 `// 停用时的处理` |

### iframe 路径规则

在 Markdown 中嵌入动画时，**必须使用绝对路径**：

```html
<!-- ✅ 正确 -->
<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/[子项目]/animations/xxx.html" ...></iframe>

<!-- ❌ 错误 -->
<iframe src="animations/xxx.html" ...></iframe>
```

### 动画开发规范
- 纯 HTML/CSS/JavaScript，无外部依赖
- 自包含单文件，可独立运行
- 提供播放/暂停/重置控制
- 清晰的中文标签和说明

## Git 工作流

### 分支
- 当前分支：`main`

### 提交规范
- 使用中文提交信息
- 清晰描述修改内容
- 每次提交是一个逻辑完整的改动

### 忽略文件
- `/.idea` - JetBrains IDE 配置
- `.DS_Store` - macOS 系统文件

## 子项目管理

### 文件夹命名规范
**遵循 macOS 文件夹命名规范**：
- 单词首字母大写（Title Case）
- 示例：`Compose/`、`Flutter/`、`Android/`

### 创建新子项目
1. 创建子项目目录：`mkdir [ProjectName]`（首字母大写）
2. 创建子项目 `CLAUDE.md`（参考现有子项目）
3. 创建 `README.md` 和 `PLAN.md`
4. 更新本文件的子项目列表

### 进入子项目工作
1. 首先阅读子项目的 `CLAUDE.md`
2. 查看 `PLAN.md` 了解进度和计划
3. 遵循子项目的特定规范

## AI 助手须知

### 工作方式
1. **查看子项目指南**：每个子项目有独立的 `CLAUDE.md`
2. **跟踪进度**：使用子项目的 `PLAN.md`
3. **保持一致**：遵循子项目的写作风格和结构

### 路径约束
- 知识库路径固定为 `/Users/tyrionguan/Documents/Obsidian Vault/AIBrain/`
- 如需移动仓库，需要批量更新所有 Markdown 文件中的 iframe 路径

---

*此文件为知识库的通用指南，各子项目有独立的 `CLAUDE.md` 提供专属指导*
