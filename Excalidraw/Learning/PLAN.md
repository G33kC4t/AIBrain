# Excalidraw Skill 创建计划

## 项目目标

在 Obsidian 中学习使用 Excalidraw，并总结为可复用的 Claude Code Skill。

## 参考资源

| 资源 | 路径/位置 | 用途 |
|------|----------|------|
| 示例文件 | `Excalidraw/Drawing 2026-02-05 20.25.39.excalidraw.md` | 了解 Obsidian Excalidraw 文件格式 |
| API 文档 | `/Users/tyrionguan/CodeLibrary/Github/excalidraw/dev-docs` | Excalidraw 官方 API |
| 现有 Skill | `/Users/tyrionguan/CodeLibrary/Github/cc-excalidraw-skill` | 参考 Skill 结构 |
| 插件 Wiki | [GitHub Wiki](https://github.com/zsviczian/obsidian-excalidraw-plugin/wiki) | 脚本引擎和使用方式 |

## 已收集的关键信息

### 1. Obsidian Excalidraw 文件格式

- 文件扩展名：`.excalidraw.md`
- 包含 YAML frontmatter（`excalidraw-plugin: parsed`）
- 文本元素提取到 `## Text Elements` 部分
- JSON 数据在 `## Drawing` 代码块中

### 2. 核心 JSON 结构

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://github.com/zsviczian/obsidian-excalidraw-plugin/...",
  "elements": [...],
  "appState": {...},
  "files": {}
}
```

### 3. 元素类型

| 类型 | 用途 |
|------|------|
| rectangle | 矩形、容器、卡片 |
| ellipse | 圆形、椭圆、节点 |
| diamond | 菱形、决策点 |
| text | 文本标签 |
| arrow | 箭头连接 |
| line | 线条连接 |
| freedraw | 手绘路径 |
| frame | 分组框架 |
| image | 嵌入图片 |

### 4. 脚本引擎功能

- 布局排列：固定间距、展开矩形
- 颜色样式：亮度调节、透明度修改
- 连接组织：自动连接元素、添加边框
- 文本处理：对齐、字体切换、分割
- 文件集成：创建并嵌入 Markdown

## 实施步骤

### 阶段 1：文件结构设计 ✅ 完成

- [x] 分析参考资源
- [x] 确定 Skill 文件结构
- [x] 创建 SKILL.md 主文件

### 阶段 2：核心内容编写 ✅ 完成

- [x] Obsidian Excalidraw 文件格式说明
- [x] 元素类型完整参考 → `element-reference.md`
- [x] 颜色和样式指南
- [x] 图表模式（流程图、架构图、时序图等）→ `diagram-patterns.md`

### 阶段 3：Obsidian 特定内容 ✅ 完成

- [x] 文本提取规则（在 SKILL.md 中）
- [x] 链接和嵌入语法（在 SKILL.md 中）
- [x] 脚本引擎使用指南（在 SKILL.md 中）
- [x] 常用脚本示例（在 SKILL.md 中）

### 阶段 4：示例和最佳实践 ✅ 完成

- [x] 完整 JSON 示例 → `examples.md`
- [x] 常见图表模板 → `examples.md`
- [x] 设计最佳实践 → `best-practices.md`

## 当前进度

**状态**：✅ 全部完成

## 创建的文件

| 文件 | 说明 |
|------|------|
| `SKILL.md` | 主 Skill 文件，包含 Obsidian Excalidraw 使用的完整指南 |
| `element-reference.md` | 所有元素类型的完整属性参考 |
| `diagram-patterns.md` | 流程图、时序图、架构图等专业图表模式 |
| `best-practices.md` | 设计最佳实践和检查清单 |
| `examples.md` | 完整可用的 JSON 示例 |

## 下一步（可选）

- [x] 将 Skill 注册到 Claude Code 配置 ✅
- [ ] 添加更多特定场景的示例
- [ ] 收集实际使用中的反馈并优化

## Skill 安装位置

已安装到 Claude Code skills 目录：
```
~/.claude/skills/learned/excalidraw/obsidian-excalidraw.md
```

## 文件结构

```
Excalidraw/
├── Drawing 2026-02-05 20.25.39.excalidraw.md  # 示例文件
├── Skills/                                      # Skill 文档
│   ├── SKILL.md                                 # 主 Skill（已安装）
│   ├── element-reference.md                     # 元素参考
│   ├── diagram-patterns.md                      # 图表模式
│   ├── best-practices.md                        # 最佳实践
│   └── examples.md                              # 示例
└── Learning/                                    # 学习记录
    └── PLAN.md                                  # 本文件
```

---

*最后更新：2026-02-05*
