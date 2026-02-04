# SlotTable 的结构 - Compose 组合树的存储原理

## 引言

在 Jetpack Compose 中，当我们编写 `@Composable` 函数时，Compose 并不是直接将这些函数转换为 UI 元素，而是经历了一个完整的转换链条：

```
@Composable 源码 → Compose Compiler 转换 → SlotTable 存储 → LayoutNode 树
```

**SlotTable** 是这个链条的核心——它是 Compose 运行时用于存储组合树结构和数据的核心数据结构。理解 SlotTable 的结构设计，是深入掌握 Compose 工作原理的基础。

### 本文聚焦

本文将深入讲解 SlotTable 的**静态结构**，包括：
- groups 和 slots 双数组的设计
- Group 的 5 个 Int 字段含义
- GroupInfo 位标记与四种 Group 类型
- Anchor 锚点机制简介

> 💡 **注意**：本文专注于 SlotTable 的数据结构，不涉及重组时的变化逻辑。重组相关内容将在第二篇《SlotTable 与重组》中详细讲解。

### 为什么需要 SlotTable？

在传统的 Android View 系统中，UI 树是由 View 对象直接构成的。每个 View 持有自己的状态、子 View 引用和绑定数据。这种设计存在几个问题：

1. **内存开销大**：每个 View 对象包含大量字段，即使是简单的 TextView 也有数百个字节
2. **更新效率低**：需要遍历整棵树来找到变化的节点
3. **状态管理复杂**：状态分散在各个 View 中，难以统一管理

Compose 采用了不同的设计思路：

| 传统 View 系统 | Compose |
|--------------|---------|
| View 对象直接构成 UI 树 | SlotTable 存储组合树，LayoutNode 构成 UI 树 |
| 状态分散在各个 View 中 | 状态集中存储在 SlotTable 的 slots 数组中 |
| 对象引用构成父子关系 | 整数索引构成父子关系 |
| 更新需要遍历对象树 | 更新通过数组索引直接定位 |

SlotTable 的设计核心思想是：**用紧凑的数组存储替代分散的对象引用**，从而实现更高效的内存使用和更新性能。

---

## 从 @Composable 到 SlotTable

### 一个简单的示例

让我们从一个简单的 Counter 组件开始：

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

这段代码看起来简洁直观，但在编译和运行时会经历复杂的转换。

### Compose Compiler 的转换

Compose Compiler 是一个 Kotlin 编译器插件，它在编译期对 `@Composable` 函数进行字节码转换：

1. **添加隐式参数**：每个 Composable 函数都会被添加 `$composer` 和 `$changed` 参数
2. **插入组管理代码**：在函数体中插入 `startGroup`/`endGroup` 调用
3. **生成跳过逻辑**：生成参数比较代码，支持智能跳过（本文不详述，见第二篇）

**编译前：**

```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello, $name!")
}
```

**编译后（简化）：**

```kotlin
fun Greeting(
    name: String,
    $composer: Composer,
    $changed: Int
) {
    $composer.startRestartGroup(123)  // 开始一个可重启组

    Text("Hello, $name!", $composer, 0)

    $composer.endRestartGroup()?.updateScope { composer, _ ->
        Greeting(name, composer, $changed or 0b0001)
    }
}
```

### Counter 编译后的调用序列

回到 Counter 示例，编译后会生成以下 `startGroup`/`endGroup` 调用序列：

> 💡 **说明**：以下为简化的伪代码，展示 Group 的嵌套结构。实际编译结果会更复杂，包含跳过逻辑等。

```kotlin
fun Counter($composer: Composer, $changed: Int) {
    $composer.startRestartGroup(key1)               // Group 0: Counter

        $composer.startReplaceableGroup(key2)       // Group 1: remember
        val count = remember { mutableStateOf(0) }  // 简化：实际会调用 cache 相关方法
        $composer.endReplaceableGroup()

        $composer.startNode()                       // Group 2: Layout（Column 内部）

            $composer.startNode()                   // Group 3: Layout（Text 内部）
            $composer.endNode()

            $composer.startRestartGroup(key4)       // Group 4: Button

                $composer.startNode()               // Group 5: Layout（Text 内部）
                $composer.endNode()

            $composer.endRestartGroup()

        $composer.endNode()

    $composer.endRestartGroup()
}
```

**关键观察：**

- 每个 Composable 调用都会创建一个对应的 **Group**
- 不同的 Composable 使用不同类型的 Group（Restartable、Replaceable、Node 等）
- Group 形成嵌套结构，反映了 Composable 的调用层次

这些 Group 最终都会被存储到 SlotTable 中。接下来，让我们深入了解 SlotTable 的核心数据结构。

---

## SlotTable 数据结构详解

理解了 Group 的概念后，我们来看 SlotTable 如何存储这些 Group。

### 双数组设计

SlotTable 的核心是两个数组：

```kotlin
class SlotTable : CompositionData {
    /**
     * 存储 Group 结构信息的数组
     * 每个 Group 占用 5 个 Int 位置
     */
    internal var groups = IntArray(0)

    /**
     * 存储实际数据的数组
     * 保存 remember 的值、节点对象、参数等
     */
    internal var slots = Array<Any?>(0) { null }

    /** 当前 Group 数量 */
    internal var groupsSize = 0

    /** 当前 Slot 数量 */
    internal var slotsSize = 0
}
```

这种**双数组分离设计**将结构信息（groups）和数据信息（slots）分开存储：

```
┌─────────────────────────────────────────────────────────────┐
│                        SlotTable                            │
├─────────────────────────────────────────────────────────────┤
│  groups (IntArray)                                          │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐    │
│  │ G0  │ G0  │ G0  │ G0  │ G0  │ G1  │ G1  │ G1  │ ... │    │
│  │key  │info │par  │size │data │key  │info │par  │     │    │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘    │
│    ↑                       ↑                                │
│    │                       │                                │
│    Group 0 (5 个 Int)      指向 slots                        │
├─────────────────────────────────────────────────────────────┤
│  slots (Array<Any?>)                                        │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┐        │
│  │ State   │ Layout  │ "Count" │ lambda  │  ...    │        │
│  │ 对象     │ Node    │ 字符串   │ 对象     │         │        │
│  └─────────┴─────────┴─────────┴─────────┴─────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**设计优势：**

| 优势       | 说明                                |
| -------- | --------------------------------- |
| **内存紧凑** | groups 使用 IntArray，每个 Int 只占 4 字节 |
| **缓存友好** | 连续的数组布局提高 CPU 缓存命中率               |
| **独立扩容** | 两个数组可以独立扩容，避免不必要的拷贝               |
| **快速定位** | 通过索引直接访问，O(1) 时间复杂度               |

### Group 的 5 个 Int 字段

每个 Group 在 groups 数组中占用 **5 个连续的 Int**：

```kotlin
// Group 字段偏移量
private const val Key_Offset = 0        // 字段 0: key
private const val GroupInfo_Offset = 1  // 字段 1: groupInfo
private const val Parent_Offset = 2     // 字段 2: parentAnchor
private const val Size_Offset = 3       // 字段 3: size
private const val DataAnchor_Offset = 4 // 字段 4: dataAnchor

private const val Group_Fields_Size = 5 // 每个 Group 占 5 个 Int
```

**字段详解：**

#### 字段 0: Key（组的唯一标识）

```
┌────────────────────────────────────────────────────────────────┐
│                           Key (32 bits)                        │
├────────────────────────────────────────────────────────────────┤
│                    完整的 key 值 (32 bits)                      │
│              通常是 Composable 调用位置的哈希值                   │
└────────────────────────────────────────────────────────────────┘
```

Key 是一个完整的 32 位整数，用于**唯一标识** Group。它的值通常由 Compose Compiler 在编译期根据 Composable 的调用位置生成。

**Key 的作用：**
- 在重组时**识别和匹配** Group
- 相同位置的 Composable 调用会生成相同的 key
- 配合 `key()` 函数可以自定义 key 值

#### 字段 1: GroupInfo（组信息）

```
┌────────────────────────────────────────────────────────────────┐
│                        GroupInfo (32 bits)                     │
├──────────────────────────────────────────────────────┬─────────┤
│           节点数量 / 数据索引 (高位)                    │ 类型标志 │
│                                                      │ (低位)  │
└──────────────────────────────────────────────────────┴─────────┘
```

GroupInfo 是最复杂的字段，它的低位存储了**类型标志位**，高位存储了节点数量或数据索引。下一节将详细讲解这些标志位。

#### 字段 2: ParentAnchor（父组锚点）

存储父 Group 的索引（或锚点），用于向上遍历组合树。

#### 字段 3: Size（组大小）

当前 Group 的大小，包含自身和所有子 Group 的总数量。用于快速跳过整个子树。

#### 字段 4: DataAnchor（数据锚点）

指向 slots 数组中该 Group 数据的起始位置。通过这个锚点可以定位到 Group 关联的数据。

### SlotTable 结构可视化

下面的动画展示了 Counter 组件在 SlotTable 中的完整存储结构，包括 groups 数组和 slots 数组的对应关系：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/group-tree-storage.html" width="100%" height="800" frameborder="0" style="border: 1px solid #ddd; border-radius: 8px; margin: 20px 0;"></iframe>

---

## GroupInfo 位标记详解

了解了 Group 的 5 个字段后，我们来深入分析最复杂的 GroupInfo 字段。它的低位存储了多个标志位，用于标识 Group 的类型和状态。

### GroupInfo 位布局可视化

下面的交互动画展示了 GroupInfo 的完整位布局结构，包括各标志位的含义和位操作演示：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/group-info-types.html" width="100%" height="800" frameborder="0" style="border: 1px solid #ddd; border-radius: 8px; margin: 20px 0;"></iframe>

### 位标记常量

> 💡 **说明**：以下常量名和位分配为示意，便于理解原理。实际源码中的命名和位分配请参考 [SlotTable.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SlotTable.kt)。

```kotlin
// GroupInfo 位标记（示意）
private const val NodeBit = 0b0001           // Bit 0: 是否为节点组
private const val ObjectKeyBit = 0b0010      // Bit 1: 是否有对象 key
private const val AuxBit = 0b0100            // Bit 2: 是否有辅助数据
private const val MarkBit = 0b1000           // Bit 3: 标记位（用于遍历）
private const val ContainMarkBit = 0b10000   // Bit 4: 子树是否包含标记

// 节点数量存储在高位
private const val NodeCountShift = 5         // 节点数量的位移量
private const val NodeCountMask = 0x7FFFFFE0.toInt() // 节点数量的掩码
```

**位标记含义：**

| 位      | 名称             | 含义                                    |
| ------ | -------------- | ------------------------------------- |
| Bit 0  | NodeBit        | 是否为**节点组**（对应 LayoutNode）             |
| Bit 1  | ObjectKeyBit   | 是否有**对象 key**（如 `key(item.id)` 中的 id） |
| Bit 2  | AuxBit         | 是否有**辅助数据**（额外存储的数据）                  |
| Bit 3  | MarkBit        | **标记位**（用于遍历时标记已访问的组）                 |
| Bit 4  | ContainMarkBit | **子树包含标记**（优化遍历，快速跳过无标记的子树）           |
| Bit 5+ | NodeCount      | **节点数量**（该 Group 子树中的 LayoutNode 总数）  |

### 读取和设置标志位

```kotlin
// 读取 GroupInfo 原始值
private fun groupInfo(groupIndex: Int): Int =
    groups[groupIndex * Group_Fields_Size + GroupInfo_Offset]

// 读取各标志位
private fun isNode(groupIndex: Int): Boolean =
    groupInfo(groupIndex) and NodeBit != 0

private fun hasObjectKey(groupIndex: Int): Boolean =
    groupInfo(groupIndex) and ObjectKeyBit != 0

private fun hasAux(groupIndex: Int): Boolean =
    groupInfo(groupIndex) and AuxBit != 0

// 设置 GroupInfo
private fun updateGroupInfo(groupIndex: Int, value: Int) {
    groups[groupIndex * Group_Fields_Size + GroupInfo_Offset] = value
}
```

### GroupInfo 高位的用途

GroupInfo 的高位（除了低位标志位）存储的是**节点数量**（nodeCount）。这个值表示该 Group 子树中包含的 LayoutNode 数量，用于快速计算和跳过。

```kotlin
// 获取节点数量（高位）
private fun nodeCount(groupIndex: Int): Int =
    groupInfo(groupIndex) shr NodeCountShift

// 设置节点数量
private fun updateNodeCount(groupIndex: Int, count: Int) {
    val info = groupInfo(groupIndex)
    val newInfo = (info and NodeCountMask.inv()) or (count shl NodeCountShift)
    updateGroupInfo(groupIndex, newInfo)
}
```

---

## 四种 Group 类型

GroupInfo 中的标志位决定了 Group 的类型。根据创建方式和用途，Group 可以分为四种类型：

### 1. Node Group（节点组）

**特征**：`isNode = true`

```kotlin
// 创建方式
$composer.startNode()
ComposeNode<LayoutNode, Applier<LayoutNode>>(
    factory = { LayoutNode() },
    update = { /* 更新属性 */ }
)
$composer.endNode()
```

**用途**：
- 对应实际的 UI 节点（LayoutNode）
- 用于 `Layout`、`Box`、`Column`、`Row` 等布局组件
- slots 中存储对应的 LayoutNode 对象

**示例**：`Column`、`Box`、`BasicText` 的内部实现

### 2. Restartable Group（可重启组）

**特征**：由 `startRestartGroup`/`endRestartGroup` 创建

```kotlin
// 创建方式（编译器自动生成）
$composer.startRestartGroup(key)
// Composable 函数体...
$composer.endRestartGroup()?.updateScope { composer, _ ->
    // 重组时的回调
    MyComposable(composer, changed)
}
```

**用途**：
- 用于普通的 `@Composable` 函数
- 支持**独立重组**：当状态变化时，可以只重组这个 Group 而不影响其他部分
- `endRestartGroup` 返回 `ScopeUpdateScope`，用于注册重组回调

**示例**：大多数自定义 Composable 函数

### 3. Replaceable Group（可替换组）

**特征**：由 `startReplaceableGroup`/`endReplaceableGroup` 创建

```kotlin
// 创建方式（编译器为 inline 函数或简单 Composable 生成）
$composer.startReplaceableGroup(key)
// 内容...
$composer.endReplaceableGroup()
```

**用途**：
- 用于 `inline` Composable 函数
- 用于不需要独立重组能力的简单组件
- 相比 Restartable Group 开销更小

**示例**：`remember`、`key`、简单的 inline Composable

### 4. Movable Group（可移动组）

**特征**：由 `startMovableGroup`/`endMovableGroup` 创建，有**对象 key**

```kotlin
// 创建方式
items.forEach { item ->
    key(item.id) {  // key() 函数会创建 Movable Group
        $composer.startMovableGroup(key, item.id)  // item.id 是对象 key
        ItemContent(item)
        $composer.endMovableGroup()
    }
}
```

**用途**：
- 用于 `key()` 函数包裹的内容
- 支持**基于 key 的重排序**：当列表顺序变化时，可以移动 Group 而不是删除重建
- `hasObjectKey = true`，对象 key 存储在 slots 中

**示例**：`LazyColumn` 中的列表项、带 `key()` 的 `forEach`

### 类型对比表

| 类型 | 创建方式 | isNode | hasObjectKey | 主要用途 |
|------|---------|--------|--------------|---------|
| Node Group | `startNode()` | ✅ | ❌ | 对应 LayoutNode |
| Restartable Group | `startRestartGroup()` | ❌ | ❌ | 普通 Composable，支持独立重组 |
| Replaceable Group | `startReplaceableGroup()` | ❌ | ❌ | inline Composable，轻量级 |
| Movable Group | `startMovableGroup()` | ❌ | ✅ | key() 包裹的内容，支持移动 |

### Counter 示例中的 Group 类型

> 💡 **说明**：以下为简化示意。实际上 `Column`、`Text` 等组件内部会调用 `Layout`，真正的 Node Group 是在 `Layout` 内部创建的。

```kotlin
@Composable
fun Counter() {
    // Group 0: Restartable Group (Counter 函数本身)

    var count by remember {  // Group 1: Replaceable Group (remember)
        mutableStateOf(0)
    }

    Column {  // 内部调用 Layout，创建 Node Group
        // Group 2: Node Group (Column 的 Layout)

        Text("Count: $count")
        // Group 3: Node Group (Text 的 Layout)

        Button(onClick = { count++ }) {
            // Group 4: Restartable Group (Button)

            Text("Increment")
            // Group 5: Node Group (Text 的 Layout)
        }
    }
}
```

---

## Slots 数组与数据存储

前面我们了解了 groups 数组如何存储 Group 的结构信息，现在来看 slots 数组如何存储实际数据。

### Slots 的用途

slots 数组存储 Group 关联的实际数据：

```kotlin
internal var slots = Array<Any?>(0) { null }
```

**存储的数据类型：**

| 数据类型 | 说明 | 示例 |
|---------|------|------|
| remember 的值 | `remember { }` 计算的结果 | `MutableState` 对象 |
| 节点对象 | Node Group 对应的节点 | `LayoutNode` |
| 对象 key | Movable Group 的 key | `item.id` |
| 辅助数据 | 额外存储的数据 | `CompositionLocal` 值 |
| 函数参数 | Composable 的参数值 | 用于比较是否变化 |

### 数据定位

每个 Group 通过 `dataAnchor` 字段指向它在 slots 中的数据起始位置：

```kotlin
// 获取 Group 的数据起始位置
private fun dataAnchor(groupIndex: Int): Int =
    groups[groupIndex * Group_Fields_Size + DataAnchor_Offset]

// 获取 Group 的数据结束位置（下一个 Group 的 dataAnchor）
private fun dataEndAnchor(groupIndex: Int): Int =
    if (groupIndex + 1 < groupsSize)
        dataAnchor(groupIndex + 1)
    else
        slotsSize
```

### Counter 示例的 Slots 内容

下图展示了 Counter 组件中各 Group 与 slots 数组的对应关系。注意 `dataAnchor` 指向的是该 Group **自身数据**的起始位置：

```
groups 数组:                          slots 数组:
┌─────────────────────┐              ┌─────────────────────────┐
│ Group 0: Counter    │──dataAnchor─→│ [0] RecomposeScope      │←─ 重组回调
│   (Restartable)     │              │                         │
├─────────────────────┤              ├─────────────────────────┤
│ Group 1: remember   │──dataAnchor─→│ [1] MutableState<Int>   │←─ remember 的值
│   (Replaceable)     │              │     (count 状态对象)     │
├─────────────────────┤              ├─────────────────────────┤
│ Group 2: Column     │──dataAnchor─→│ [2] LayoutNode (Column) │←─ Column 节点
│   (Node)            │              │                         │
├─────────────────────┤              ├─────────────────────────┤
│ Group 3: Text       │──dataAnchor─→│ [3] LayoutNode (Text)   │←─ Text 节点
│   (Node)            │              │ [4] "Count: 0"          │←─ 文本内容
├─────────────────────┤              ├─────────────────────────┤
│ Group 4: Button     │──dataAnchor─→│ [5] LayoutNode (Button) │←─ Button 节点
│   (Node)            │              │ [6] onClick lambda      │←─ 点击回调
├─────────────────────┤              ├─────────────────────────┤
│ Group 5: Text       │──dataAnchor─→│ [7] LayoutNode (Text)   │←─ Text 节点
│   (Node)            │              │ [8] "Increment"         │←─ 文本内容
└─────────────────────┘              └─────────────────────────┘
```

> 💡 **说明**：实际的 slots 内容会更复杂，可能包含 Modifier、CompositionLocal 等额外数据。上图为简化示意。

---

## Anchor 机制简介

在 SlotTable 的设计中，**Anchor（锚点）** 是一个重要的辅助机制，用于在 SlotTable 结构发生变化时保持对特定位置的稳定引用。

### 为什么需要 Anchor？

SlotTable 使用 Gap Buffer 机制（将在第二篇详细讲解）来高效支持插入和删除操作。当数据插入或删除时，数组中的索引位置会发生变化。如果外部代码直接持有数组索引，这些索引在数据变化后就会失效。

**Anchor 解决的问题**：提供一个**稳定的引用**，即使 SlotTable 内部结构变化，Anchor 仍能正确指向目标 Group。

### Anchor 的工作原理

```kotlin
class Anchor(internal var location: Int) {
    val valid: Boolean get() = location != Int.MIN_VALUE
}
```

Anchor 内部持有一个 `location` 值：
- 当 SlotTable **没有**在写入时，`location` 直接表示 Group 索引
- 当 SlotTable **正在**写入时，`location` 会被转换为相对于 Gap 的位置

SlotTable 维护了一个 Anchor 列表，在写入操作完成后会**批量更新**所有 Anchor 的位置，确保它们始终指向正确的 Group。

### Anchor 的用途

| 用途 | 说明 |
|------|------|
| **ParentAnchor 字段** | Group 的第 3 个字段就是用 Anchor 机制指向父 Group |
| **DataAnchor 字段** | Group 的第 5 个字段用 Anchor 机制指向 slots 数组位置 |
| **跨组合周期引用** | 外部代码可以通过 Anchor 持有对某个 Group 的稳定引用 |
| **CompositionLocal** | 用于追踪 Provider 的位置 |

### 示例：Anchor 如何保持稳定

```
初始状态：
groups: [G0] [G1] [G2] [G3]
        ↑
     Anchor(location=0) 指向 G0

插入新 Group 后：
groups: [G0] [Gnew] [G1] [G2] [G3]
        ↑
     Anchor(location=0) 仍然指向 G0 ✓

如果直接用索引：
index = 1 原本指向 G1
插入后 index = 1 指向 Gnew ✗ （错误！）
```

> 💡 **详细内容预告**：Anchor 与 Gap Buffer 的协作机制将在第二篇《SlotTable 与重组》中深入讲解。

---

## 从 SlotTable 到 LayoutNode 树（简介）

SlotTable 存储了完整的组合树结构，但它并不是最终的 UI 树。Compose 需要将 SlotTable 中的 **Node Group** 转换为 **LayoutNode** 树。

### Node Group 与 LayoutNode 的对应关系

只有 `isNode = true` 的 Group 才对应一个 LayoutNode：

```
SlotTable (groups 数组)              LayoutNode 树
━━━━━━━━━━━━━━━━━━━━━━              ━━━━━━━━━━━━━━━
Group 0: Counter (Restartable)
    ↓
Group 1: remember (Replaceable)      (无对应节点)
    ↓
Group 2: Column (Node) ──────────→  LayoutNode (Column)
    ↓                                    │
Group 3: Text (Node)                     ├─ LayoutNode (Text)
    ↓                                    │
Group 4: Button (Node) ──────────→       └─ LayoutNode (Button)
    ↓                                          │
Group 5: Text (Node)                           └─ LayoutNode (Text)
```

### Applier 的基本作用

**Applier** 负责将 SlotTable 的变化应用到 LayoutNode 树。当组合完成后，Compose 会调用 Applier 来：

1. **插入节点**：当有新的 Node Group 时，创建并插入 LayoutNode
2. **删除节点**：当 Node Group 被移除时，删除对应的 LayoutNode
3. **移动节点**：当 Node Group 位置变化时，移动 LayoutNode

```kotlin
abstract class AbstractApplier<T>(val root: T) : Applier<T> {
    abstract fun insertTopDown(index: Int, instance: T)
    abstract fun insertBottomUp(index: Int, instance: T)
    abstract fun remove(index: Int, count: Int)
    abstract fun move(from: Int, to: Int, count: Int)
}
```

> 💡 **详细内容预告**：Applier 和 applyChanges 的详细机制将在第三篇《SlotTable 到 LayoutNode》中深入讲解。

---

## 总结

本文详细讲解了 SlotTable 的静态结构，这是理解 Compose 运行时的基础。

### 核心要点回顾

**1. 双数组设计**
- `groups`：IntArray，存储 Group 的结构信息，每个 Group 占用 5 个 Int
- `slots`：Array<Any?>，存储实际数据（状态、节点、参数等）

**2. Group 的 5 个字段**

| 字段 | 用途 |
|------|------|
| Key | 组的唯一标识，用于重组时匹配 |
| GroupInfo | 类型标志位 + 节点数量 |
| ParentAnchor | 父组引用，支持向上遍历 |
| Size | 组大小（含子组），支持快速跳过 |
| DataAnchor | 指向 slots 数组中的数据位置 |

**3. 四种 Group 类型**

| 类型 | 特征 | 用途 |
|------|------|------|
| Node Group | isNode=true | 对应 LayoutNode |
| Restartable Group | 支持独立重组 | 普通 Composable |
| Replaceable Group | 轻量级 | inline Composable |
| Movable Group | hasObjectKey=true | key() 包裹的内容 |

**4. Anchor 机制**
- 提供稳定的引用，不受 SlotTable 结构变化影响
- 用于 ParentAnchor、DataAnchor 等字段
- 支持跨组合周期的 Group 引用

### 下一篇预告

在下一篇《SlotTable 与重组》中，我们将深入讲解：
- Gap Buffer 机制：如何高效支持插入和删除
- SlotReader 与 SlotWriter：读写 SlotTable 的核心类
- 重组时 SlotTable 的变化：数据更新、结构变化、差异计算

---

## 参考资源

- [Compose Runtime 源码](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [SlotTable.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SlotTable.kt)
- [Compose 官方文档：Understanding Composition](https://developer.android.com/jetpack/compose/mental-model)
