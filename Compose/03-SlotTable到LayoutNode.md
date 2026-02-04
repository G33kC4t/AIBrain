# SlotTable 到 LayoutNode - applyChanges 的秘密

## 引言

在前两篇文章中，我们深入了解了 SlotTable 的静态结构和动态变化机制。但 SlotTable 本身只是一个**数据容器**——它存储了 UI 的结构信息，却不能直接显示到屏幕上。

真正呈现在用户面前的是 **LayoutNode 树**——一棵由 LayoutNode 节点组成的树形结构，每个节点对应屏幕上的一个 UI 元素。

那么问题来了：**SlotTable 中的数据如何转换成 LayoutNode 树？**

答案就是本文的主角——**applyChanges 机制**。

### Compose 渲染的三个阶段

在深入 applyChanges 之前，让我们先建立整体认知。Compose 渲染一帧分为三个阶段：

```
┌─────────────────────────────────────────────────────────────┐
│                    Compose 渲染流水线                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  数据（State）                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Composition │ ─→ │   Layout    │ ─→ │   Drawing   │     │
│  │   组合阶段   │    │   布局阶段   │    │   绘制阶段   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│       │                   │                   │            │
│       ▼                   ▼                   ▼            │
│  "显示什么"           "放在哪里"          "如何渲染"          │
│  执行 @Composable     测量和放置          绘制到 Canvas      │
│  生成 UI 树描述        确定位置和大小       呈现到屏幕         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

本文聚焦于**组合阶段**的最后一步：将 SlotTable 中收集的变化应用到 LayoutNode 树。

### 本文聚焦

本文将深入讲解：
- LayoutNode 树的结构与作用
- Changes 的收集机制
- Applier 的设计与实现
- **applyChanges 的完整流程**（重点）
- ComposeNode 的工作原理

> 💡 **系列文章导航**：
> - 第一篇：[SlotTable 的结构](01-SlotTable的结构.md) - 静态数据结构
> - 第二篇：[SlotTable 与重组](02-SlotTable与重组.md) - Gap Buffer 与动态变化
> - 第三篇：SlotTable 到 LayoutNode（本文）- applyChanges 机制
> - 第四篇：SubcomposeLayout 与复用机制 - 延迟组合与复用

---

## 两阶段设计：组合与应用分离

在讲解具体机制之前，我们需要理解一个关键的设计决策：**为什么要分离组合和应用？**

### 为什么不直接修改 LayoutNode？

一个直观的想法是：在执行 @Composable 函数时，直接创建或修改 LayoutNode。但 Compose 选择了不同的方式：

```
直接修改方式（未采用）：
┌───────────────────────────────────────┐
│ @Composable 执行                       │
│     ↓                                 │
│ 立即修改 LayoutNode 树                  │
│     ↓                                 │
│ 可能产生不一致的中间状态                 │
└───────────────────────────────────────┘

两阶段方式（实际采用）：
┌───────────────────────────────────────┐
│ 阶段1：组合                            │
│ @Composable 执行 → 收集 Changes        │
│ （SlotTable 更新，但 LayoutNode 未变）  │
│                                       │
│ 阶段2：应用                            │
│ applyChanges() → 批量修改 LayoutNode   │
│ （一次性应用所有变化）                   │
└───────────────────────────────────────┘
```

### 两阶段设计的优势

**1. 一致性保证**

```kotlin
// 假设有这样的代码
@Composable
fun Parent() {
    val state = remember { mutableStateOf(0) }
    Child1(state.value)  // 读取 state
    state.value++        // 修改 state（假设允许）
    Child2(state.value)  // 再次读取 state
}
```

如果直接修改 LayoutNode，Child1 和 Child2 会看到不同的 state 值，导致 UI 不一致。

两阶段设计确保：**所有 Changes 收集完成后，一次性应用**，避免中间状态。

**2. 性能优化**

```
批量操作 vs 逐个操作：

逐个操作（10 次插入）：
insert → 布局 → insert → 布局 → ... (10 次布局计算)

批量操作：
收集 10 个 insert → 一次性应用 → 1 次布局计算
```

**3. 事务语义**

类似数据库事务，组合阶段的所有修改要么全部成功应用，要么全部回滚（如果出错）。

---

## LayoutNode 树结构

### 什么是 LayoutNode？

**LayoutNode** 是 Compose UI 的核心节点类型，代表 UI 树中的一个元素。每个你在屏幕上看到的 UI 组件（Text、Button、Column 等）最终都对应一个 LayoutNode。

```kotlin
// LayoutNode 的核心属性（简化）
class LayoutNode {
    // 树结构
    var parent: LayoutNode? = null
    val children: MutableList<LayoutNode> = mutableListOf()

    // 测量策略（定义如何测量和放置子节点）
    var measurePolicy: MeasurePolicy = ...

    // Modifier 链（定义外观和行为）
    var modifier: Modifier = Modifier

    // 布局信息
    var width: Int = 0
    var height: Int = 0
    var x: Int = 0
    var y: Int = 0

    // 绘制方法
    fun draw(canvas: Canvas) { ... }
}
```

### LayoutNode 树 vs SlotTable

SlotTable 和 LayoutNode 树是**两个不同的数据结构**，它们之间存在映射关系：

```
SlotTable 中的 Group 结构：

[Root Group]
├── [Restartable: MyScreen]
│   ├── [Replaceable: Column]        ← 不是 Node Group
│   │   ├── [Node: Text]             ← isNode=true, 对应 LayoutNode
│   │   ├── [Node: Button]           ← isNode=true, 对应 LayoutNode
│   │   │   └── [Node: Text]         ← isNode=true, 对应 LayoutNode
│   │   └── [Restartable: Counter]
│   │       └── [Node: Text]         ← isNode=true, 对应 LayoutNode
```

```
对应的 LayoutNode 树：

[Root LayoutNode]
└── [LayoutNode: Column]
    ├── [LayoutNode: Text]
    ├── [LayoutNode: Button]
    │   └── [LayoutNode: Text]
    └── [LayoutNode: Text]
```

**关键对应规则**：
- 只有 **isNode=true** 的 Group 才对应 LayoutNode
- Restartable Group、Replaceable Group 等不产生 LayoutNode
- Group 的嵌套关系决定 LayoutNode 的父子关系

### 示例：代码到 LayoutNode

```kotlin
@Composable
fun Greeting(name: String) {
    Column {                              // LayoutNode: Column
        Text("Hello, $name!")             // LayoutNode: Text
        Button(onClick = { }) {           // LayoutNode: Button
            Text("Click me")              // LayoutNode: Text (Button 的子节点)
        }
    }
}
```

生成的 LayoutNode 树：

```
LayoutNode (Column)
├── LayoutNode (Text: "Hello, $name!")
└── LayoutNode (Button)
    └── LayoutNode (Text: "Click me")
```

---

## Changes 收集机制

在组合阶段，Composer 遍历 SlotTable 并执行 @Composable 函数。当检测到需要修改 LayoutNode 树时，它不会立即修改，而是**收集 Change 对象**。

### Change 的类型

Compose 定义了多种 Change 类型来描述对 LayoutNode 树的操作：

```kotlin
// Change 的概念模型（简化的伪代码，展示核心逻辑）
// 实际实现使用 lambda 形式，而非 sealed class
sealed class Change {
    // 记住一个值（存储在 SlotTable）
    class Remember(val value: Any?) : Change()

    // 更新槽位值
    class UpdateValue(val index: Int, val value: Any?) : Change()

    // 插入节点
    class InsertNode(val index: Int, val node: LayoutNode) : Change()

    // 移除节点
    class RemoveNode(val index: Int, val count: Int) : Change()

    // 移动节点
    class MoveNode(val from: Int, val to: Int, val count: Int) : Change()

    // 更新节点属性
    class UpdateNode(val node: LayoutNode, val block: () -> Unit) : Change()
}
```

### Changes 的收集时机

Changes 在**组合阶段**被收集，存储在一个列表中：

```kotlin
// Composition 内部（简化的伪代码，展示核心逻辑）
class CompositionImpl {
    private val changes = mutableListOf<Change>()

    // 组合时调用
    fun compose() {
        composer.startComposition()

        // 执行 @Composable 函数
        // 期间会调用 recordChange() 收集变化
        content()

        composer.endComposition()
    }

    // 记录一个 Change
    internal fun recordChange(change: Change) {
        changes.add(change)
    }
}
```

### 示例：首次组合时的 Changes

```kotlin
@Composable
fun SimpleUI() {
    Column {
        Text("Hello")
    }
}
```

首次组合时收集的 Changes：

```
Changes 列表：
1. InsertNode(index=0, node=LayoutNode<Column>)
2. InsertNode(index=0, node=LayoutNode<Text>)  // Column 的子节点
3. UpdateNode(node=Text节点, block={ set text = "Hello" })
```

### 示例：重组时的 Changes

假设从 `Text("Hello")` 变为 `Text("World")`：

```
Changes 列表：
1. UpdateNode(node=Text节点, block={ set text = "World" })

注意：没有 InsertNode 或 RemoveNode，因为结构未变
```

### 示例：条件变化时的 Changes

```kotlin
@Composable
fun Conditional(showA: Boolean) {
    if (showA) {
        Text("A")
    } else {
        Text("B")
    }
}
```

从 `showA=true` 变为 `showA=false` 时：

```
Changes 列表：
1. RemoveNode(index=0, count=1)     // 移除 Text("A")
2. InsertNode(index=0, node=Text)   // 插入 Text("B")
3. UpdateNode(node=Text, block={ set text = "B" })
```

---

## Applier 机制

### 什么是 Applier？

**Applier** 是一个接口，定义了如何将 Changes 应用到目标树结构。它是 Compose Runtime 与具体平台（Android、Desktop、Web）之间的桥梁。

```kotlin
// Applier 接口定义
interface Applier<N> {
    // 当前操作的节点
    val current: N

    // 导航方法
    fun down(node: N)    // 进入子节点
    fun up()             // 返回父节点

    // 节点操作
    fun insertTopDown(index: Int, instance: N)    // 自顶向下插入
    fun insertBottomUp(index: Int, instance: N)   // 自底向上插入
    fun remove(index: Int, count: Int)            // 移除节点
    fun move(from: Int, to: Int, count: Int)      // 移动节点

    // 清空所有子节点
    fun clear()
}
```

### insertTopDown vs insertBottomUp

这两个方法的区别在于**插入的顺序**：

**insertTopDown**：先插入父节点，再插入子节点
```
插入顺序：Parent → Child1 → Child2
          ↓
       [Parent]
       /      \
  [Child1]  [Child2]
```

**insertBottomUp**：先创建子节点，再将父节点与子节点关联
```
创建顺序：Child1 → Child2 → Parent（关联子节点）
                              ↓
                           [Parent]
                           /      \
                      [Child1]  [Child2]
```

**为什么需要两种方式？**

不同的树结构可能有不同的最优插入顺序：

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| Android View | insertBottomUp | 子 View 添加到 parent 时触发 layout |
| LayoutNode | insertBottomUp | 避免多次触发测量 |
| 某些自定义树 | insertTopDown | 父节点需要先存在 |

### UiApplier 实现

在 Compose UI 中，**UiApplier** 是 Applier 的具体实现，负责操作 LayoutNode 树：

```kotlin
// UiApplier 的简化实现（展示核心逻辑）
class UiApplier(root: LayoutNode) : AbstractApplier<LayoutNode>(root) {

    // 自底向上插入（Compose UI 使用这种方式）
    override fun insertBottomUp(index: Int, instance: LayoutNode) {
        current.insertAt(index, instance)
    }

    // 不使用自顶向下插入
    override fun insertTopDown(index: Int, instance: LayoutNode) {
        // 空实现，因为使用 insertBottomUp
    }

    // 移除节点
    override fun remove(index: Int, count: Int) {
        current.removeAt(index, count)
    }

    // 移动节点
    override fun move(from: Int, to: Int, count: Int) {
        current.move(from, to, count)
    }

    // 清空所有子节点
    override fun onClear() {
        current.removeAll()
    }
}
```

### AbstractApplier 基类

AbstractApplier 提供了导航功能的默认实现：

```kotlin
abstract class AbstractApplier<N>(val root: N) : Applier<N> {
    // 节点栈，用于追踪当前位置
    private val stack = mutableListOf<N>()

    // 当前节点
    override var current: N = root
        protected set

    // 进入子节点
    override fun down(node: N) {
        stack.add(current)
        current = node
    }

    // 返回父节点
    override fun up() {
        current = stack.removeLast()
    }

    // 重置到根节点
    protected fun clear() {
        stack.clear()
        current = root
    }
}
```

### Applier 导航示例

假设要在以下树结构中的 Column 下插入新节点：

```
Root
└── Column      ← 目标父节点
    ├── Text
    └── Button
```

Applier 的操作序列：

```kotlin
// 1. 当前在 Root
applier.current  // = Root

// 2. 进入 Column
applier.down(column)
applier.current  // = Column

// 3. 在 Column 下插入新节点
applier.insertBottomUp(2, newNode)  // 在索引 2 插入

// 4. 返回 Root
applier.up()
applier.current  // = Root
```

---

## applyChanges 详解

现在我们来到本文的核心：**applyChanges** 函数。它负责将收集的所有 Changes 批量应用到 LayoutNode 树。

下面的交互式动画展示了 applyChanges 在三种场景下的完整执行流程：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/apply-changes-flow.html" width="100%" height="700" frameborder="0" style="border: 1px solid #333; border-radius: 8px; margin: 20px 0;"></iframe>

### applyChanges 的调用时机

applyChanges 在**组合完成后、布局之前**被调用：

```
┌─────────────────────────────────────────────────────────┐
│                    一帧的执行流程                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Choreographer.doFrame()                                │
│       │                                                 │
│       ├── 1. 处理输入事件                                │
│       │                                                 │
│       ├── 2. 执行动画回调                                │
│       │                                                 │
│       ├── 3. Compose 组合 ←────────────────────┐        │
│       │       │                               │        │
│       │       ├── 执行 @Composable 函数        │        │
│       │       ├── 更新 SlotTable              │        │
│       │       ├── 收集 Changes               │        │
│       │       └── applyChanges() ◀━━━ 这里！  │        │
│       │                                       │        │
│       ├── 4. 布局（测量 + 放置）               │        │
│       │                                       │        │
│       └── 5. 绘制                             │        │
│                                               │        │
│  如果有无效化 ────────────────────────────────┘        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### applyChanges 的执行流程

```kotlin
// Composition 中的 applyChanges（简化的伪代码，展示核心逻辑）
fun applyChanges() {
    // 1. 获取 Applier
    val applier = this.applier

    // 2. 遍历并执行所有 Changes
    for (change in changes) {
        change.apply(applier, slotTable)
    }

    // 3. 清空 Changes 列表
    changes.clear()

    // 4. 处理副作用（如 LaunchedEffect、DisposableEffect）
    applyLateChanges()
}
```

### Change 的执行顺序

Changes 按照**收集的顺序**执行，这个顺序与 @Composable 函数的执行顺序一致：

```kotlin
@Composable
fun Parent() {
    Child1()  // Change 1, 2, 3...
    Child2()  // Change 4, 5, 6...
    Child3()  // Change 7, 8, 9...
}

// Changes 执行顺序：1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9
```

### 详细示例：插入节点

假设首次组合以下代码：

```kotlin
@Composable
fun SimpleList() {
    Column {
        Text("Item 1")
        Text("Item 2")
    }
}
```

**收集的 Changes**：

```
1. [Navigate] down(root)
2. [Insert]   insertBottomUp(0, LayoutNode<Column>)
3. [Navigate] down(column)
4. [Insert]   insertBottomUp(0, LayoutNode<Text1>)
5. [Update]   updateNode(Text1) { text = "Item 1" }
6. [Insert]   insertBottomUp(1, LayoutNode<Text2>)
7. [Update]   updateNode(Text2) { text = "Item 2" }
8. [Navigate] up()
9. [Navigate] up()
```

**执行过程**：

```
初始状态：
[Root] (current = Root)

执行 Change 1 (down):
[Root] ← current
   └── (empty)

执行 Change 2 (insert Column):
[Root]
   └── [Column]    ← 新插入

执行 Change 3 (down Column):
[Root]
   └── [Column] ← current

执行 Change 4 (insert Text1):
[Root]
   └── [Column] ← current
       └── [Text1]    ← 新插入

执行 Change 5 (update Text1):
[Root]
   └── [Column]
       └── [Text1: "Item 1"]    ← 更新文本

执行 Change 6 (insert Text2):
[Root]
   └── [Column]
       ├── [Text1: "Item 1"]
       └── [Text2]    ← 新插入

执行 Change 7 (update Text2):
[Root]
   └── [Column]
       ├── [Text1: "Item 1"]
       └── [Text2: "Item 2"]    ← 更新文本

执行 Change 8, 9 (up x2):
[Root] ← current（回到根节点）
   └── [Column]
       ├── [Text1: "Item 1"]
       └── [Text2: "Item 2"]
```

### 详细示例：删除节点

假设从显示两个 Text 变为只显示一个：

```kotlin
// Before
@Composable
fun Content() {
    Text("Keep")
    Text("Remove")  // 这个将被移除
}

// After
@Composable
fun Content() {
    Text("Keep")
    // Text("Remove") 被移除
}
```

**收集的 Changes**：

```
1. [Skip]   skipGroup(Text "Keep")   // 未变化，跳过
2. [Remove] remove(index=1, count=1) // 移除第二个节点
```

**执行过程**：

```
初始状态：
[Parent]
├── [Text: "Keep"]
└── [Text: "Remove"]

执行 remove(1, 1):
[Parent]
└── [Text: "Keep"]

Text "Remove" 的 LayoutNode 被移除
```

### 详细示例：移动节点

假设列表顺序从 [A, B, C] 变为 [C, A, B]：

```kotlin
// 使用 key() 确保可以移动而非重建
items.forEach { item ->
    key(item) {
        ItemCard(item)
    }
}
```

**收集的 Changes**：

```
1. [Move] move(from=2, to=0, count=1)  // 将 C 从索引 2 移动到索引 0
// A, B 自动顺移，无需额外操作
```

**执行过程**：

```
初始状态：
[Parent]
├── [ItemCard: A]  index=0
├── [ItemCard: B]  index=1
└── [ItemCard: C]  index=2

执行 move(2, 0, 1):
[Parent]
├── [ItemCard: C]  index=0  ← 移动到这里
├── [ItemCard: A]  index=1  ← 原来的 0，现在是 1
└── [ItemCard: B]  index=2  ← 原来的 1，现在是 2
```

**移动的优势**：
- LayoutNode 实例保持不变
- 所有内部状态（remember、动画等）保留
- 比删除+重建更高效

---

## ComposeNode 的工作原理

在实际代码中，LayoutNode 的创建和更新是通过 **ComposeNode** 函数完成的。

### ComposeNode 函数签名

```kotlin
@Composable
inline fun <T : Any, reified E : Applier<*>> ComposeNode(
    noinline factory: () -> T,           // 创建节点的工厂函数
    update: @DisallowComposableCalls Updater<T>.() -> Unit  // 更新节点的 lambda
)
```

### factory 块：节点创建

**factory** 块在**首次组合**时执行，用于创建新的节点实例：

```kotlin
@Composable
fun BasicText(text: String) {
    ComposeNode<LayoutNode, UiApplier>(
        factory = {
            // 只在首次组合时执行
            LayoutNode().apply {
                // 初始化测量策略等
                measurePolicy = TextMeasurePolicy()
            }
        },
        update = { /* ... */ }
    )
}
```

### update 块：节点更新

**update** 块在**每次组合**时都会执行，用于更新节点属性：

```kotlin
@Composable
fun BasicText(text: String) {
    ComposeNode<LayoutNode, UiApplier>(
        factory = { LayoutNode() },
        update = {
            // 每次组合都执行
            // 通过 set() 函数更新属性
            set(text) { node ->
                // 只有当 text 值变化时才真正更新
                node.textContent = text
            }
        }
    )
}
```

### Updater 的智能更新

Updater 提供了智能的属性更新机制：

```kotlin
// Updater 的简化实现（展示核心逻辑）
// 实际实现通过 SlotTable 的 slots 数组存储旧值
class Updater<N>(private val composer: Composer, private val node: N) {
    // 只有值变化时才更新
    inline fun <V> set(value: V, crossinline block: N.(V) -> Unit) {
        val oldValue = composer.rememberedValue()
        if (oldValue !== value) {
            composer.updateRememberedValue(value)
            // 记录一个 UpdateNode Change
            composer.recordChange {
                node.block(value)
            }
        }
    }

    // 无论是否变化都更新
    inline fun <V> update(value: V, crossinline block: N.(V) -> Unit) {
        composer.updateRememberedValue(value)
        composer.recordChange {
            node.block(value)
        }
    }
}
```

### 实际示例：Text 的 ComposeNode

```kotlin
// Text 内部实现（简化的伪代码，展示核心逻辑）
@Composable
fun Text(
    text: String,
    modifier: Modifier = Modifier,
    style: TextStyle = LocalTextStyle.current
) {
    // 获取必要的上下文
    val density = LocalDensity.current

    ComposeNode<LayoutNode, UiApplier>(
        factory = {
            // 创建 LayoutNode
            LayoutNode()
        },
        update = {
            // 更新文本内容
            set(text) { this.textContent = it }

            // 更新样式
            set(style) { this.textStyle = it }

            // 更新 Modifier
            set(modifier) { this.modifier = it }

            // 更新测量策略
            set(density) { density ->
                this.measurePolicy = TextMeasurePolicy(density)
            }
        }
    )
}
```

### Modifier 的应用过程

Modifier 是一个链式结构，在 update 块中被应用到节点：

```kotlin
// Modifier 链的构建
val modifier = Modifier
    .padding(16.dp)
    .background(Color.White)
    .clickable { }

// update 块中应用 Modifier
set(modifier) { newModifier ->
    // LayoutNode 内部会：
    // 1. 对比新旧 Modifier
    // 2. 只更新变化的部分
    // 3. 重新构建 ModifierNode 链
    this.modifier = newModifier
}
```

Modifier 的更新策略：

| 情况 | 处理方式 |
|------|---------|
| Modifier 相同 | 跳过，无操作 |
| Modifier 链长度相同 | 逐个对比，只更新变化的节点 |
| Modifier 链长度不同 | 重建整个 ModifierNode 链 |

---

## applyChanges 完整流程图

让我们用一个完整的流程图总结 applyChanges 的工作机制：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     applyChanges 完整流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     1. 组合阶段                                │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                                                               │  │
│  │  @Composable 函数执行                                         │  │
│  │       │                                                       │  │
│  │       ├── SlotWriter 遍历 SlotTable                           │  │
│  │       │                                                       │  │
│  │       ├── 检测变化：                                          │  │
│  │       │   ├── 新 Group → 生成 InsertNode Change               │  │
│  │       │   ├── Group 消失 → 生成 RemoveNode Change             │  │
│  │       │   ├── Group 移动 → 生成 MoveNode Change               │  │
│  │       │   └── 属性变化 → 生成 UpdateNode Change               │  │
│  │       │                                                       │  │
│  │       └── Changes 收集到列表                                   │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   2. applyChanges()                           │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                                                               │  │
│  │  遍历 Changes 列表                                             │  │
│  │       │                                                       │  │
│  │       ├── InsertNode:                                         │  │
│  │       │   └── applier.insertBottomUp(index, node)             │  │
│  │       │                                                       │  │
│  │       ├── RemoveNode:                                         │  │
│  │       │   └── applier.remove(index, count)                    │  │
│  │       │                                                       │  │
│  │       ├── MoveNode:                                           │  │
│  │       │   └── applier.move(from, to, count)                   │  │
│  │       │                                                       │  │
│  │       └── UpdateNode:                                         │  │
│  │           └── 执行更新 lambda                                  │  │
│  │                                                               │  │
│  │  清空 Changes 列表                                             │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   3. LayoutNode 树更新完成                     │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                                                               │  │
│  │  LayoutNode 树现在反映了最新的 UI 结构                          │  │
│  │       │                                                       │  │
│  │       ├── 新节点已添加                                        │  │
│  │       ├── 删除的节点已移除                                    │  │
│  │       ├── 移动的节点已重排                                    │  │
│  │       └── 属性已更新                                          │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   4. 进入布局阶段                              │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                                                               │  │
│  │  LayoutNode 树被测量和放置                                      │  │
│  │  每个节点确定 width, height, x, y                              │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 总结

本文深入讲解了 SlotTable 数据如何转换为 LayoutNode 树的完整机制。

### 核心概念回顾

**1. 两阶段设计**
- 组合阶段：执行 @Composable，收集 Changes
- 应用阶段：applyChanges() 批量更新 LayoutNode 树
- 优势：一致性保证、性能优化、事务语义

**2. LayoutNode 树**
- Compose UI 的核心节点类型
- 只有 isNode=true 的 Group 对应 LayoutNode
- 包含 parent、children、modifier、measurePolicy 等属性

**3. Changes 机制**
- InsertNode：插入新节点
- RemoveNode：移除节点
- MoveNode：移动节点
- UpdateNode：更新节点属性

**4. Applier**
- 接口：定义如何操作目标树
- UiApplier：Compose UI 的具体实现
- 导航：down()、up() 遍历树
- 操作：insertBottomUp()、remove()、move()

**5. ComposeNode**
- factory 块：首次组合创建节点
- update 块：每次组合更新属性
- Updater：智能更新，只更新变化的属性

### 性能要点

| 操作类型 | 性能影响 | 优化建议 |
|---------|---------|---------|
| 数据更新 | 最低 | 优先使用，避免结构变化 |
| 节点移动 | 中等 | 使用 key() 启用移动优化 |
| 节点插入/删除 | 较高 | 减少条件分支切换 |
| 完全重建 | 最高 | 避免大范围结构变化 |

### 下一篇预告

在第四篇《SubcomposeLayout 与复用机制》中，我们将讲解：
- SubcomposeLayout 的使用与原理
- Subcomposition 的生命周期
- SlotTable 与 LayoutNode 的复用机制
- LazyColumn 的高效列表实现

---

## 参考资源

- [Compose Runtime 源码](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [Applier.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Applier.kt)
- [Composition.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composition.kt)
- [Compose Phases 官方文档](https://developer.android.com/develop/ui/compose/phases)
- [Compose Internals Book](https://jorgecastillo.dev/book/)
