# SlotTable 与重组 - 动态变化的秘密

## 引言

在上一篇文章中，我们了解了 SlotTable 的静态结构：groups 数组存储 Group 的结构信息，slots 数组存储实际数据。但 SlotTable 的真正威力在于它如何高效地支持**动态变化**——这正是重组（Recomposition）的核心需求。

当 UI 状态发生变化时，Compose 需要：
1. 在 SlotTable 中插入新的 Group（如条件分支切换、列表项增加）
2. 删除不再需要的 Group（如条件分支切换、列表项减少）
3. 更新现有 Group 的数据（如状态值变化）


如果使用普通数组，每次插入或删除都需要移动大量数据，复杂度为 O(n)。为了解决这个问题，SlotTable 使用了一种经典的数据结构——**Gap Buffer**。
### 本文聚焦

本文将深入讲解：
- Gap Buffer 的设计原理及其在 SlotTable 中的应用
- SlotReader 与 SlotWriter 的工作机制
- 三种典型重组场景下的 SlotTable 变化
- Key 匹配算法与跳过机制

> 💡 **系列文章导航**：
> - 第一篇：[SlotTable 的结构](01-SlotTable的结构.md) - 静态数据结构
> - 第二篇：SlotTable 与重组（本文）- Gap Buffer 与动态变化
> - 第三篇：SlotTable 到 LayoutNode - applyChanges 机制

---

## Gap Buffer 设计原理

### 什么是 Gap Buffer？

**Gap Buffer** 是一种专门为"局部编辑"优化的数据结构，最早应用于文本编辑器。它的核心思想是：

> 在数组中维护一段连续的空白区域（Gap），所有的插入和删除操作都发生在 Gap 的边界。

```
传统数组插入：需要移动后面所有元素
[A][B][C][D][E]  → 在 B 后插入 X →  [A][B][X][C][D][E]
           ↑                              ↑
       移动 3 个元素                    O(n) 复杂度

Gap Buffer 插入：只需在 Gap 边界写入
[A][B][   GAP   ][C][D][E]  → 在 Gap 起始位置写入 X
                ↓
[A][B][X][  GAP ][C][D][E]  O(1) 复杂度！
```

Gap Buffer 特别适合以下场景：
- **编辑操作具有局部性**：用户通常在某个位置连续编辑
- **插入/删除频繁**：频繁的数组操作会带来大量数据移动
- **数据量较大**：数据量越大，O(n) 和 O(1) 的差距越明显

这正是 Compose 重组的典型场景！

### 为什么 SlotTable 使用 Gap Buffer？

在 Compose 的重组过程中，SlotWriter 按照 Composable 的执行顺序遍历 SlotTable：

1. **局部性强**：重组通常只影响部分 UI，变化的 Group 往往集中在某个区域
2. **连续操作**：一个 Composable 函数内部的多次插入/删除是连续的
3. **性能敏感**：重组可能在每一帧都发生，必须足够快

使用 Gap Buffer 后：
- 连续插入：第一次移动 Gap，后续插入都是 O(1)
- 连续删除：通过扩展 Gap "吞入"数据，O(1) 完成
- 跳过不变的部分：Gap 不需要移动

---

## SlotTable 中的双 Gap 机制

SlotTable 实际上维护了**两个独立的 Gap**——一个用于 groups 数组，一个用于 slots 数组。

### Groups Gap

```kotlin
// groups 数组的 Gap 状态
internal var groups = IntArray(0)
internal var groupsSize = 0      // 有效 Group 数量（不含 Gap）

// Gap 位置信息（在 SlotWriter 中）
private var groupGapStart = 0    // Gap 起始位置（Group 索引）
private var groupGapLen = 0      // Gap 长度（Group 数量）
```

Groups 数组的逻辑布局：

```
groups 数组物理布局：
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ G0  │ G1  │ GAP │ GAP │ GAP │ G2  │ G3  │ G4  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
索引:  0     1     2     3     4     5     6     7

groupGapStart = 2
groupGapLen = 3
groupsSize = 5 (有效 Group: G0, G1, G2, G3, G4)

逻辑视图（Gap 不可见）：
┌─────┬─────┬─────┬─────┬─────┐
│ G0  │ G1  │ G2  │ G3  │ G4  │
└─────┴─────┴─────┴─────┴─────┘
逻辑索引: 0     1     2     3     4
```

### Slots Gap

```kotlin
// slots 数组的 Gap 状态
internal var slots = Array<Any?>(0) { null }
internal var slotsSize = 0       // 有效 Slot 数量（不含 Gap）

// Gap 位置信息（在 SlotWriter 中）
private var slotsGapStart = 0    // Gap 起始位置
private var slotsGapLen = 0      // Gap 长度
```

### 双 Gap 协同工作

Groups Gap 和 Slots Gap 是**独立管理**的，但在操作时需要**协同移动**：

```
当在 groups 中间插入新 Group 时：
1. 移动 Groups Gap 到插入位置
2. 移动 Slots Gap 到对应的 slots 位置
3. 写入新 Group 的数据（5 个 Int）
4. 写入新 Group 关联的 slots 数据
```

这种设计的优势是：
- **独立扩容**：groups 和 slots 可以独立扩容，避免不必要的内存分配
- **灵活管理**：不同 Group 可能有不同数量的 slots，独立管理更灵活

---

## Gap 操作详解

### 移动 Gap（moveGroupGapTo）

当需要在非 Gap 位置进行操作时，必须先移动 Gap。

**Gap 向前移动**（目标位置 < 当前 Gap 位置）：

```kotlin
// 简化的伪代码，展示核心逻辑
private fun moveGroupGapTo(index: Int) {
    if (index < groupGapStart) {
        // 将 [index, groupGapStart) 的数据复制到 Gap 后面
        val count = groupGapStart - index
        groups.copyInto(
            destination = groups,
            destinationOffset = (index + groupGapLen) * 5,
            startIndex = index * 5,
            endIndex = groupGapStart * 5
        )
        groupGapStart = index
    }
}
```

```
移动前：gap 在索引 4
[G0][G1][G2][G3][ GAP ][ GAP ][G4][G5]
                   ↑
              groupGapStart = 4

目标：移动 gap 到索引 1
需要将 [G1, G2, G3] 复制到 gap 后面

移动后：
[G0][ GAP ][ GAP ][G1][G2][G3][G4][G5]
        ↑
   groupGapStart = 1
```

**Gap 向后移动**（目标位置 > 当前 Gap 位置）：

```kotlin
// 简化的伪代码，展示核心逻辑
private fun moveGroupGapTo(index: Int) {
    if (index > groupGapStart) {
        // 将 (groupGapStart + groupGapLen, index + groupGapLen] 的数据复制到 Gap 前面
        groups.copyInto(
            destination = groups,
            destinationOffset = groupGapStart * 5,
            startIndex = (groupGapStart + groupGapLen) * 5,
            endIndex = (index + groupGapLen) * 5
        )
        groupGapStart = index
    }
}
```

### 在 Gap 位置插入（insertGroups）

当 Gap 已经在目标位置时，插入操作非常简单：

```kotlin
// 简化的伪代码，展示核心逻辑
fun insertGroups(count: Int): Int {
    // 确保 Gap 足够大
    if (groupGapLen < count) {
        ensureGroupGap(count)
    }

    val insertIndex = groupGapStart

    // 直接在 Gap 起始位置写入数据
    // groups[groupGapStart * 5 + Key_Offset] = key
    // groups[groupGapStart * 5 + GroupInfo_Offset] = info
    // ...

    // 更新 Gap 指针
    groupGapStart += count
    groupGapLen -= count
    groupsSize += count

    return insertIndex  // 返回插入位置
}
```

**关键点**：插入操作只需要：
1. 写入数据到 Gap 起始位置
2. 更新 Gap 指针

**没有任何数据移动**，时间复杂度 O(1)！

### 删除 Group（removeGroups）

删除操作通过**扩展 Gap** 来"吞入"要删除的数据：

```kotlin
// 简化的伪代码，展示核心逻辑
fun removeGroups(index: Int, count: Int) {
    // 移动 Gap 到删除位置的末尾
    moveGroupGapTo(index + count)

    // 向前扩展 Gap，"吞入"要删除的数据
    groupGapStart -= count
    groupGapLen += count
    groupsSize -= count
}
```

```
删除前：要删除 G2, G3（假设 Gap 在末尾）
[G0][G1][G2][G3][G4][G5][ GAP ]
                           ↑
                      gapStart = 6

步骤1：移动 Gap 到 G3 后面（index + count = 2 + 2 = 4）
[G0][G1][G2][G3][ GAP ][G4][G5]
                   ↑
              gapStart = 4

步骤2：向前扩展 Gap（gapStart -= count）
[G0][G1][     GAP      ][G4][G5]
        ↑
   gapStart = 2
   gapLen += 2

G2, G3 被 Gap "覆盖"，逻辑上已删除
```

**关键点**：删除操作只需要：
1. 移动 Gap 到删除位置末尾
2. 调整 Gap 指针

如果 Gap 已经在正确位置，删除也是 O(1)！

### 扩展 Gap（ensureGroupGap）

当 Gap 空间不足时，需要扩容：

```kotlin
// 简化的伪代码，展示核心逻辑
private fun ensureGroupGap(required: Int) {
    if (groupGapLen < required) {
        // 计算新容量（通常是 2 倍扩容）
        val oldCapacity = groups.size / 5
        val newCapacity = maxOf(oldCapacity * 2, oldCapacity + required)

        // 分配新数组
        val newGroups = IntArray(newCapacity * 5)

        // 复制 Gap 前的数据
        groups.copyInto(
            newGroups,
            destinationOffset = 0,
            startIndex = 0,
            endIndex = groupGapStart * 5
        )

        // 复制 Gap 后的数据到新数组末尾
        val newGapLen = newCapacity - groupsSize
        groups.copyInto(
            newGroups,
            destinationOffset = groupGapStart * 5 + newGapLen * 5,
            startIndex = groupGapStart * 5 + groupGapLen * 5,
            endIndex = groups.size
        )

        groups = newGroups
        groupGapLen = newGapLen
    }
}
```

---

## Gap Buffer 操作可视化

下面的交互式动画展示了 Gap Buffer 在三种复杂场景下的工作过程，你可以清楚地看到代码执行与数据变化的对应关系：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/gap-buffer-operations.html" width="100%" height="850" frameborder="0" style="border: 1px solid #333; border-radius: 8px; margin: 20px 0;"></iframe>

**三个场景说明**：

| 场景 | 操作 | 复杂度分析 |
|------|------|-----------|
| 中间插入 | 在非 Gap 位置插入新 Group | O(n) 移动 + O(1) 写入 |
| 批量删除 | 删除连续多个 Group | O(n) 移动 Gap + O(1) 扩展 |
| 移动后插入 | 移动一次后连续插入多个 | O(n) 移动 + k × O(1) 写入 |

---

## Gap Buffer 的性能优势

### 对比：有 Gap Buffer vs 无 Gap Buffer

假设 SlotTable 中有 1000 个 Group，需要在中间位置连续插入 10 个新 Group：

**无 Gap Buffer（普通数组）**：
```
第 1 次插入：移动 500 个元素
第 2 次插入：移动 501 个元素
第 3 次插入：移动 502 个元素
...
第 10 次插入：移动 509 个元素

总计：移动约 5045 个元素
```

**有 Gap Buffer**：
```
移动 Gap 到插入位置：移动 500 个元素（一次性）
第 1 次插入：O(1)
第 2 次插入：O(1)
...
第 10 次插入：O(1)

总计：移动约 500 个元素
```

**性能提升**：约 10 倍！

### 最佳实践：利用局部性

Gap Buffer 的性能优势来自于**操作的局部性**。Compose 的设计天然符合这一特点：

1. **Composable 函数的执行顺序**是确定的，对 SlotTable 的访问是线性的
2. **状态变化通常是局部的**，只影响部分 UI
3. **条件分支切换**导致的插入/删除是连续的

```kotlin
@Composable
fun MyList(items: List<Item>) {
    Column {
        items.forEach { item ->
            // 每个 item 的 Group 是连续存储的
            // 如果 items 变化，插入/删除操作也是连续的
            ItemCard(item)
        }
    }
}
```

---

## Gap Buffer 小结

本节我们深入讲解了 Gap Buffer 机制：

### 核心要点

**1. Gap Buffer 原理**
- 在数组中维护一段连续的空白区域（Gap）
- 插入/删除操作发生在 Gap 边界
- 利用操作的局部性，减少数据移动

**2. SlotTable 的双 Gap**
- Groups Gap：管理 groups 数组
- Slots Gap：管理 slots 数组
- 两个 Gap 独立管理，协同工作

**3. 关键操作**
| 操作 | 方法 | 复杂度 |
|------|------|--------|
| 移动 Gap | `moveGroupGapTo()` | O(n) |
| 插入 | `insertGroups()` | O(1)* |
| 删除 | `removeGroups()` | O(1)* |
| 扩容 | `ensureGroupGap()` | O(n) |

*假设 Gap 已在正确位置

### 下一节预告

在下一节中，我们将讲解 **SlotReader 与 SlotWriter**：
- SlotReader 如何读取 SlotTable 数据
- SlotWriter 的插入模式与更新模式
- Reader/Writer 的协作机制

---

## SlotReader 与 SlotWriter

SlotTable 本身只是一个数据容器，真正的读写操作通过两个专门的类来完成：**SlotReader** 和 **SlotWriter**。这种设计实现了读写分离，提供了更好的并发安全性和更清晰的 API。

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        SlotTable                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  groups: IntArray      slots: Array<Any?>           │   │
│  └─────────────────────────────────────────────────────┘   │
│                    │                    │                   │
│         ┌─────────┴────────┐  ┌────────┴─────────┐        │
│         ▼                  ▼  ▼                  ▼        │
│   ┌───────────┐      ┌───────────┐      ┌───────────┐     │
│   │SlotReader │      │SlotReader │      │SlotWriter │     │
│   │  (只读)   │      │  (只读)   │      │ (读写)    │     │
│   └───────────┘      └───────────┘      └───────────┘     │
│         ▲                  ▲                  ▲            │
│         │                  │                  │            │
│    可同时存在           可同时存在         同时只能有一个     │
│    多个 Reader          多个 Reader           Writer       │
└─────────────────────────────────────────────────────────────┘
```

### SlotReader：只读遍历器

SlotReader 提供对 SlotTable 的**只读访问**，用于遍历和读取 Group 数据。

#### 核心状态

```kotlin
class SlotReader(
    internal val table: SlotTable
) {
    // 当前遍历位置
    var currentGroup: Int = 0        // 当前 Group 索引
        private set
    var currentSlot: Int = 0         // 当前 Slot 索引
        private set

    // 父 Group 栈（用于嵌套遍历）
    private var parent: Int = -1     // 当前父 Group
    private val groupEnds = IntStack() // 记录各层父 Group 的结束位置

    // 从 SlotTable 复制的数组引用（只读）
    private val groups: IntArray = table.groups
    private val slots: Array<Any?> = table.slots
}
```

#### 主要操作

**1. 遍历 Group**

```kotlin
// 开始遍历一个 Group（进入子 Group）
fun startGroup() {
    // 记录当前 Group 作为父 Group
    groupEnds.push(currentGroup + groupSize(currentGroup))
    parent = currentGroup
    currentGroup++
    currentSlot = dataAnchor(currentGroup)
}

// 结束当前 Group 的遍历
fun endGroup() {
    // 跳过未遍历的子 Group
    currentGroup = groupEnds.pop()
    parent = if (groupEnds.isNotEmpty()) groupEnds.peek() else -1
}

// 跳过当前 Group 及其所有子 Group
fun skipGroup(): Int {
    val count = groupSize(currentGroup)
    currentGroup += count
    currentSlot = dataAnchor(currentGroup)
    return count
}
```

**2. 读取 Group 信息**

```kotlin
// 读取当前 Group 的 key
val groupKey: Int get() = groups[currentGroup * 5 + Key_Offset]

// 读取当前 Group 的大小（含子 Group）
val groupSize: Int get() = groups[currentGroup * 5 + Size_Offset]

// 当前 Group 是否是节点组
val isNode: Boolean get() = groupInfo(currentGroup) and NodeBit != 0

// 当前 Group 是否有对象 key
val hasObjectKey: Boolean get() = groupInfo(currentGroup) and ObjectKeyBit != 0

// 当前 Group 的子 Group 数量
val childCount: Int get() {
    // 计算方式：groupSize - 1（自身）- 所有子 Group 的 size
    // 实际实现更复杂，需要遍历子 Group
}
```

**3. 读取 Slot 数据**

```kotlin
// 读取下一个 slot 值
fun next(): Any? {
    val result = slots[currentSlot]
    currentSlot++
    return result
}

// 读取当前 Group 的节点对象（Node Group 专用）
fun node(): Any? {
    return if (isNode) slots[dataAnchor(currentGroup)] else null
}

// 读取对象 key（Movable Group 专用）
fun objectKey(): Any? {
    return if (hasObjectKey) slots[dataAnchor(currentGroup) + nodeOffset] else null
}
```

#### Reader 使用示例

```kotlin
// 遍历 SlotTable 中的所有 Group
fun dumpSlotTable(table: SlotTable) {
    val reader = table.openReader()
    try {
        while (reader.currentGroup < reader.groupsSize) {
            val key = reader.groupKey
            val size = reader.groupSize
            val isNode = reader.isNode

            println("Group $key: size=$size, isNode=$isNode")

            if (reader.hasChildren) {
                reader.startGroup()
                // 递归遍历子 Group...
                reader.endGroup()
            } else {
                reader.skipGroup()
            }
        }
    } finally {
        reader.close()
    }
}
```

---

### SlotWriter：读写操作器

SlotWriter 提供对 SlotTable 的**读写访问**，负责在组合和重组过程中修改 SlotTable。

#### 核心状态

```kotlin
class SlotWriter(
    internal val table: SlotTable
) {
    // 继承 Reader 的遍历状态
    var currentGroup: Int = 0
    var currentSlot: Int = 0
    private var parent: Int = -1

    // Gap Buffer 状态
    private var groupGapStart: Int = 0
    private var groupGapLen: Int = 0
    private var slotsGapStart: Int = 0
    private var slotsGapLen: Int = 0

    // 插入模式标志
    var inserting: Boolean = false
        private set

    // 可变数组引用
    private var groups: IntArray = table.groups
    private var slots: Array<Any?> = table.slots
}
```

#### 两种工作模式

SlotWriter 有两种截然不同的工作模式：

**1. 插入模式（inserting = true）**

用于**首次组合**，所有 Group 都是新创建的：

```kotlin
// 进入插入模式
fun beginInsert() {
    inserting = true
}

// 退出插入模式
fun endInsert() {
    inserting = false
}

// 插入模式下的 startGroup
fun startGroup(key: Int, objectKey: Any?, isNode: Boolean, aux: Any?) {
    if (inserting) {
        // 创建新 Group
        insertGroups(1)
        initGroup(key, objectKey, isNode, aux)
    }
    // ... 更新遍历状态
}
```

**2. 更新模式（inserting = false）**

用于**重组**，需要匹配现有 Group 并决定复用或替换：

```kotlin
// 更新模式下的 startGroup
fun startGroup(key: Int, objectKey: Any?, isNode: Boolean, aux: Any?) {
    if (!inserting) {
        val currentKey = groups[currentGroup * 5 + Key_Offset]
        if (currentKey == key) {
            // Key 匹配，复用现有 Group
            // 只需更新 slot 数据
        } else {
            // Key 不匹配，需要替换
            // 删除旧 Group，插入新 Group
        }
    }
    // ... 更新遍历状态
}
```

#### 主要操作

**1. Group 操作**

```kotlin
// 开始一个 Group（根据模式决定插入或复用）
fun startGroup(key: Int) {
    if (inserting) {
        insertGroups(1)
        // 写入 Group 数据...
    } else {
        // 匹配 key，决定复用或替换
    }
    advanceGroup()
}

// 结束当前 Group
fun endGroup(): Int {
    val expectedEnd = groupEnds.pop()
    if (currentGroup != expectedEnd) {
        if (inserting) {
            // 更新 parent 的 size
        } else {
            // 删除多余的 Group
            removeGroups(expectedEnd - currentGroup)
        }
    }
    parent = groupEnds.peekOr(-1)
    return currentGroup
}

// 跳过当前 Group（参数未变化时）
fun skipGroup() {
    val size = groupSize(currentGroup)
    currentGroup += size
    currentSlot = dataAnchor(currentGroup)
}
```

**2. Slot 操作**

```kotlin
// 更新 slot 值
fun update(value: Any?): Any? {
    val oldValue = slots[currentSlot]
    slots[currentSlot] = value
    currentSlot++
    return oldValue
}

// 跳过 slot（值未变化）
fun skip() {
    currentSlot++
}

// 更新 slot（仅当值变化时）
fun updateIfChanged(value: Any?): Boolean {
    val oldValue = slots[currentSlot]
    if (oldValue != value) {
        slots[currentSlot] = value
        currentSlot++
        return true
    }
    currentSlot++
    return false
}
```

**3. 节点操作**

```kotlin
// 设置当前 Group 的节点
fun setNode(node: Any?) {
    val dataIndex = dataAnchor(currentGroup)
    slots[dataIndex] = node
}

// 获取当前 Group 的节点
fun node(): Any? {
    return slots[dataAnchor(currentGroup)]
}
```

---

### Reader 与 Writer 的协作

SlotTable 的并发访问遵循以下规则：

#### 访问规则

```kotlin
class SlotTable {
    private var writer: SlotWriter? = null
    private var readers = mutableListOf<SlotReader>()

    // 获取 Reader（可以有多个）
    fun openReader(): SlotReader {
        check(writer == null) { "Cannot read while writing" }
        val reader = SlotReader(this)
        readers.add(reader)
        return reader
    }

    // 获取 Writer（只能有一个）
    fun openWriter(): SlotWriter {
        check(writer == null) { "Already writing" }
        check(readers.isEmpty()) { "Cannot write while reading" }
        writer = SlotWriter(this)
        return writer!!
    }

    // 关闭 Reader
    internal fun closeReader(reader: SlotReader) {
        readers.remove(reader)
    }

    // 关闭 Writer
    internal fun closeWriter(writer: SlotWriter) {
        // 将 Gap 移动到末尾，便于读取
        writer.moveGapToEnd()
        this.writer = null
    }
}
```

#### 为什么 Writer 关闭时要移动 Gap？

```
写入时（Gap 可能在任意位置）：
[G0][G1][ GAP ][ GAP ][G2][G3][G4]

关闭 Writer 后（Gap 移动到末尾）：
[G0][G1][G2][G3][G4][ GAP ][ GAP ]
                     ^
                     便于 Reader 线性遍历
```

这样 Reader 可以无视 Gap 的存在，直接从头到尾遍历有效数据。

---

### Composer 如何使用 Writer

在实际的组合过程中，Composer 持有 SlotWriter 并控制其工作模式：

```kotlin
class ComposerImpl(
    private val slotTable: SlotTable
) {
    private lateinit var writer: SlotWriter

    // 开始组合
    fun startComposition() {
        writer = slotTable.openWriter()
        // 首次组合：进入插入模式
        // 重组：保持更新模式
    }

    // startRestartGroup 的简化实现
    fun startRestartGroup(key: Int): Composer {
        writer.startGroup(key)
        return this
    }

    // endRestartGroup 的简化实现
    fun endRestartGroup(): ScopeUpdateScope? {
        writer.endGroup()
        // 返回重组回调注册器
        return if (needsRecompose) scopeUpdateScope else null
    }

    // 结束组合
    fun endComposition() {
        writer.close()
    }
}
```

---

### 总结：Reader 与 Writer 对比

| 特性 | SlotReader | SlotWriter |
|------|------------|------------|
| **访问模式** | 只读 | 读写 |
| **并发数量** | 可多个同时存在 | 只能有一个 |
| **Gap 感知** | 不感知（Gap 在末尾） | 完全控制 Gap |
| **使用场景** | 遍历、调试、序列化 | 组合、重组 |
| **主要方法** | `next()`, `skip()`, `startGroup()` | `update()`, `insert()`, `remove()` |

---

## 重组时 SlotTable 数据变化详解

了解了 Gap Buffer 机制和 Reader/Writer 的工作原理后，我们来看三个典型的重组场景，深入理解 SlotTable 在重组时的具体变化。

下面的交互式动画展示了三种典型重组场景下 SlotTable 的动态变化过程：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/slottable-recomposition.html" width="100%" height="750" frameborder="0" style="border: 1px solid #333; border-radius: 8px; margin: 20px 0;"></iframe>

### 场景一：数据更新（Slots 变化，Groups 不变）

这是最简单也是最常见的重组场景：状态值变化，但 UI 结构不变。

#### 示例代码

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Text("Count: $count")  // count 变化时触发重组
    Button(onClick = { count++ }) {
        Text("Increment")
    }
}
```

当 `count` 从 0 变为 1 时，会发生什么？

#### SlotTable 变化分析

```
首次组合后的 SlotTable：
┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Counter][remember][Text][Button][Text]                     │
│                                                             │
│ slots:                                                      │
│ [...][MutableState(0)][...]["Count: 0"][...]["Increment"]   │
│                              ↑                              │
│                        需要更新的值                          │
└─────────────────────────────────────────────────────────────┘

重组时：
1. SlotWriter 遍历到 Text Group
2. 发现 key 匹配，复用 Group（不修改 groups 数组）
3. 调用 writer.update("Count: 1") 更新 slot 值

重组后：
┌─────────────────────────────────────────────────────────────┐
│ groups: [不变]                                              │
│ [Counter][remember][Text][Button][Text]                     │
│                                                             │
│ slots:                                                      │
│ [...][MutableState(1)][...]["Count: 1"][...]["Increment"]   │
│                              ↑                              │
│                          值已更新                           │
└─────────────────────────────────────────────────────────────┘
```

#### 关键点

- **Groups 数组完全不变**：无需移动 Gap
- **只更新 Slots 中的值**：O(1) 操作
- **这是性能最好的重组场景**

---

### 场景二：结构变化（条件分支切换）

当 `if`/`when` 条件变化时，UI 结构会发生变化，需要删除旧 Group 并插入新 Group。

#### 示例代码

```kotlin
@Composable
fun ConditionalContent(showA: Boolean) {
    if (showA) {
        Text("Content A")  // 分支 A
        Icon(Icons.A)
    } else {
        Text("Content B")  // 分支 B
    }
}
```

当 `showA` 从 `true` 变为 `false` 时：

#### SlotTable 变化分析

```
showA = true 时的 SlotTable：
┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Parent][if-Group key=A][Text-A][Icon-A][ GAP ]            │
│          └─────────────────────────────┘                    │
│                   分支 A 的内容                              │
└─────────────────────────────────────────────────────────────┘

切换到 showA = false，重组过程：

步骤1：SlotWriter 遍历到 if-Group
       发现当前 key=A，但期望 key=B，不匹配！

步骤2：删除分支 A 的 Group
       调用 writer.removeGroups() 删除 [if-Group][Text-A][Icon-A]

┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Parent][        GAP (扩展)        ]                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

步骤3：插入分支 B 的 Group
       进入插入模式，创建新 Group

┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Parent][if-Group key=B][Text-B][ GAP ]                     │
│          └─────────────────────┘                            │
│              分支 B 的内容                                   │
└─────────────────────────────────────────────────────────────┘
```

#### Compiler 生成的 Key

Compose Compiler 会为不同的条件分支生成不同的 key：

```kotlin
// 编译后的伪代码
fun ConditionalContent(showA: Boolean, $composer: Composer) {
    if (showA) {
        $composer.startReplaceableGroup(key = 0x1a2b3c)  // 分支 A 的 key
        Text("Content A")
        Icon(Icons.A)
        $composer.endReplaceableGroup()
    } else {
        $composer.startReplaceableGroup(key = 0x4d5e6f)  // 分支 B 的 key
        Text("Content B")
        $composer.endReplaceableGroup()
    }
}
```

不同的 key 确保了分支切换时会触发 Group 替换。

---

### 场景三：列表项增删

列表操作是最复杂的场景，涉及多个 Group 的批量增删。

#### 示例代码

```kotlin
@Composable
fun ItemList(items: List<String>) {
    Column {
        items.forEach { item ->
            key(item) {  // 使用 item 作为 key
                ListItem(item)
            }
        }
    }
}
```

假设列表从 `["A", "B", "C"]` 变为 `["A", "D", "C"]`（B 被替换为 D）：

#### SlotTable 变化分析

```
初始状态 items = ["A", "B", "C"]：
┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Column][key="A"][Item-A][key="B"][Item-B][key="C"][Item-C] │
│         └──────────────────────────────────────────────────┘│
│                    三个列表项的 Group                         │
│                                                             │
│ slots:                                                      │
│ [...]["A"][...]["B"][...]["C"][...]                         │
└─────────────────────────────────────────────────────────────┘

重组 items = ["A", "D", "C"]：

步骤1：遍历到 key="A"，匹配成功，跳过

步骤2：遍历到 key="B"，但期望 key="D"
       - 检查后面是否有 key="D" 的 Group → 没有
       - 删除 key="B" 的 Group
       - 插入 key="D" 的新 Group

┌─────────────────────────────────────────────────────────────┐
│ groups:                                                     │
│ [Column][key="A"][Item-A][key="D"][Item-D][key="C"][Item-C] │
│                          └───────────────┘                  │
│                          替换后的 Group                       │
└─────────────────────────────────────────────────────────────┘

步骤3：遍历到 key="C"，匹配成功，跳过
```

#### 如果顺序变化而非内容变化？

假设从 `["A", "B", "C"]` 变为 `["C", "A", "B"]`：

```kotlin
// 使用 key() 的优势
items.forEach { item ->
    key(item) {  // Movable Group
        ListItem(item)
    }
}
```

Compose 可以**移动** Group 而不是删除重建：

```
初始：[key="A"][key="B"][key="C"]

期望：[key="C"][key="A"][key="B"]

Compose 的处理：
1. 期望 key="C"，当前 key="A"
2. 在后面找到 key="C" 的 Group
3. 移动 key="C" 到当前位置（而非删除 A 再创建 C）
4. 继续处理 key="A"、key="B"
```

**移动 vs 删除重建的性能差异**：

| 操作 | 移动 | 删除重建 |
|------|------|----------|
| Group 数据 | 保留 | 丢失后重建 |
| Slot 数据 | 保留 | 丢失后重建 |
| remember 状态 | 保留 | 重新初始化 |
| 副作用 | 不触发 | 重新执行 |

---

### 重组决策流程图

```
                    ┌─────────────────────┐
                    │ SlotWriter 遍历到   │
                    │   当前 Group        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  当前 key == 期望 key?│
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │ Yes            │                │ No
              ▼                │                ▼
    ┌─────────────────┐        │    ┌─────────────────┐
    │   复用 Group    │        │    │ 在后面查找匹配的│
    │ 只更新 slots    │        │    │   Movable Group │
    └─────────────────┘        │    └────────┬────────┘
                               │             │
                               │    ┌────────┼────────┐
                               │    │ Found  │        │ Not Found
                               │    ▼        │        ▼
                               │ ┌──────────┐│  ┌──────────┐
                               │ │移动 Group ││  │删除当前   │
                               │ │到当前位置  ││  │Group     │
                               │ └──────────┘│  │插入新     │
                               │             │  │Group     │
                               │             │  └──────────┘
                               │             │
                               └─────────────┘
```

下面的交互式动画展示了 Key 匹配算法在三种场景下的决策过程：

<iframe src="file:///Users/tyrionguan/Documents/Obsidian%20Vault/AIBrain/Compose/animations/key-matching.html" width="100%" height="800" frameborder="0" style="border: 1px solid #333; border-radius: 8px; margin: 20px 0;"></iframe>

---

### 性能优化建议

根据以上分析，我们可以得出一些优化建议：

**1. 尽量让重组只更新数据，不改变结构**

```kotlin
// 好：结构不变，只更新数据
@Composable
fun Item(data: ItemData) {
    Text(data.title)     // 只更新文本内容
    Text(data.subtitle)
}

// 避免：频繁的结构变化
@Composable
fun Item(data: ItemData) {
    if (data.hasTitle) {     // 条件可能频繁切换
        Text(data.title)
    }
    if (data.hasSubtitle) {
        Text(data.subtitle)
    }
}
```

**2. 为列表项使用稳定的 key**

```kotlin
// 好：使用唯一且稳定的 ID
items.forEach { item ->
    key(item.id) {
        ItemCard(item)
    }
}

// 避免：使用 index 作为 key
items.forEachIndexed { index, item ->
    key(index) {  // 插入/删除会导致大量 Group 替换
        ItemCard(item)
    }
}
```

**3. 将不变的部分提取出来**

```kotlin
// 好：Header 不会因为 items 变化而重组
@Composable
fun List(items: List<Item>) {
    Column {
        Header()  // 独立的 Composable，不依赖 items
        items.forEach { item ->
            ItemCard(item)
        }
    }
}
```

---

## 重组的差异计算

理解了 SlotTable 的变化过程后，让我们深入看看 Compose 是如何决定"哪些 Group 需要重组"的。

### Key 匹配算法

SlotWriter 在遍历时，通过 **Key 匹配**来决定是复用还是替换 Group。

#### 基本匹配规则

```kotlin
// SlotWriter 中的 key 匹配逻辑（简化）
fun startGroup(key: Int, objectKey: Any?) {
    if (!inserting) {
        val currentKey = groups[currentGroup * 5 + Key_Offset]

        when {
            // 情况1：key 完全匹配
            currentKey == key && matchObjectKey(objectKey) -> {
                // 复用当前 Group
                reuseGroup()
            }

            // 情况2：当前 key 不匹配，尝试查找 Movable Group
            isMovableGroup(currentGroup) -> {
                val found = findGroupWithKey(key, objectKey)
                if (found != -1) {
                    // 移动找到的 Group 到当前位置
                    moveGroup(found, currentGroup)
                } else {
                    // 删除当前 Group，插入新 Group
                    replaceGroup(key, objectKey)
                }
            }

            // 情况3：普通 Group 不匹配
            else -> {
                replaceGroup(key, objectKey)
            }
        }
    }
}
```

#### Movable Group 的查找范围

当需要查找可移动的 Group 时，Compose 会在**当前父 Group 的剩余子 Group**中查找：

```
父 Group 的子 Group 范围：
[Child-A][Child-B][Child-C][Child-D][Child-E]
          ↑                         ↑
     currentGroup              查找范围结束
                               (父 Group 结束位置)

查找范围 = [currentGroup + 1, parentEnd)
```

这意味着：
- **只能在同级 Group 中移动**
- **不能跨父 Group 移动**
- **查找是线性的**，但通常范围有限

### skipToGroupEnd 跳过机制

这是 Compose 性能优化的关键：当 Composable 的参数没有变化时，可以完全跳过该 Group 的重组。

#### 跳过条件

```kotlin
// Composer 中的跳过逻辑
fun startRestartGroup(key: Int): Composer {
    writer.startGroup(key)

    // 检查是否可以跳过
    if (!inserting && isSkipping && !hasInvalidations) {
        // 可以跳过！直接跳到 Group 末尾
        skipToGroupEnd()
        return this
    }

    // 不能跳过，需要执行组合
    return this
}
```

跳过的条件：
1. **不在插入模式**：首次组合不能跳过
2. **当前正在跳过模式**：`isSkipping = true`
3. **该 Group 没有无效化**：没有被 `invalidate()` 标记

#### $changed 参数的作用

Compose Compiler 生成的 `$changed` 参数用于追踪参数变化：

```kotlin
// 编译前
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
}

// 编译后（简化）
fun Greeting(
    name: String,
    $composer: Composer,
    $changed: Int  // 位标记，记录哪些参数变化了
) {
    $composer.startRestartGroup(key)

    // 检查参数是否变化
    val nameChanged = $changed and 0b0001 != 0

    if (!nameChanged && $composer.skipping) {
        // 参数未变化，可以跳过
        $composer.skipToGroupEnd()
    } else {
        // 参数变化，需要重新执行
        Text("Hello, $name", $composer, 0)
    }

    $composer.endRestartGroup()
}
```

#### 跳过的效果

```
不跳过（完整重组）：
┌──────────────────────────────────────────┐
│ 遍历: [G0] → [G1] → [G2] → [G3] → [G4]   │
│       执行  执行   执行   执行   执行     │
│                                          │
│ 时间: ████████████████████████████████   │
└──────────────────────────────────────────┘

跳过（智能重组）：
┌──────────────────────────────────────────┐
│ 遍历: [G0] → [G1] ──────────────→ [G4]   │
│       执行  跳过 G1 整个子树      执行    │
│                                          │
│ 时间: ████████                ████████   │
│              ↑                           │
│         节省的时间                        │
└──────────────────────────────────────────┘
```

### 无效化（Invalidation）机制

当状态变化时，Compose 需要知道哪些 Group 受影响，这通过**无效化**机制实现。

#### RecomposeScope 的注册

每个 Restartable Group 都会注册一个 `RecomposeScope`：

```kotlin
fun endRestartGroup(): ScopeUpdateScope? {
    val scope = if (inserting) {
        // 首次组合：创建新的 RecomposeScope
        RecomposeScope(currentGroup).also {
            // 存储在 slots 中
            writer.update(it)
        }
    } else {
        // 重组：读取现有的 RecomposeScope
        writer.next() as? RecomposeScope
    }

    writer.endGroup()

    // 返回 scope，用于注册重组回调
    return scope?.takeIf { it.valid }
}
```

#### 状态变化触发无效化

```kotlin
// MutableState 的实现
class SnapshotMutableStateImpl<T>(value: T) : MutableState<T> {
    override var value: T
        get() = // 读取时记录依赖
        set(newValue) {
            if (newValue != field) {
                field = newValue
                // 通知所有依赖此状态的 RecomposeScope
                Snapshot.notifyObjectsChanged()
            }
        }
}

// 无效化流程
fun invalidate(scope: RecomposeScope) {
    // 将 scope 加入待重组列表
    pendingInvalidations.add(scope)
    // 触发下一帧重组
    scheduleRecomposition()
}
```

---

## 总结

本文深入讲解了 SlotTable 在重组时的变化机制，让我们回顾核心要点：

### 核心概念回顾

**1. Gap Buffer 机制**
- 通过维护连续的空白区域（Gap）实现高效插入/删除
- groups 和 slots 各自维护独立的 Gap
- 利用操作的局部性，将 O(n) 操作降低到 O(1)

**2. SlotReader 与 SlotWriter**
- SlotReader：只读遍历，可多个同时存在
- SlotWriter：读写操作，只能有一个
- 插入模式（首次组合）vs 更新模式（重组）

**3. 三种重组场景**

| 场景 | Groups 变化 | Slots 变化 | 复杂度 |
|------|-------------|------------|--------|
| 数据更新 | 不变 | 更新值 | O(1) |
| 结构变化 | 删除 + 插入 | 对应变化 | O(n) |
| 列表操作 | 可能移动 | 对应变化 | O(n) ~ O(1) |

**4. 差异计算**
- Key 匹配：决定复用、移动还是替换
- 跳过机制：参数未变化时跳过整个子树
- 无效化：精确追踪哪些 Group 需要重组

### 性能优化要点

1. **尽量保持结构稳定**：避免条件分支频繁切换
2. **使用稳定的 key**：列表项使用唯一 ID 而非 index
3. **细粒度状态**：让状态变化只影响最小范围
4. **利用跳过机制**：确保 Composable 参数是 Stable 的

### 下一篇预告

在第三篇《SlotTable 到 LayoutNode》中，我们将讲解：
- applyChanges 机制：SlotTable 变化如何同步到 UI 树
- Applier 的工作原理
- Changes 的收集与应用

---

## 参考资源

- [Compose Runtime 源码](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/)
- [SlotTable.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SlotTable.kt)
- [Composer.kt](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composer.kt)
- [Compose 官方文档：Lifecycle](https://developer.android.com/jetpack/compose/lifecycle)
- [深入理解 Compose 重组](https://developer.android.com/jetpack/compose/mental-model)
