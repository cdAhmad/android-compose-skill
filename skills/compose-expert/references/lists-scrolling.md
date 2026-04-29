# Jetpack Compose 中的列表与滚动

高效的列表渲染和滚动是响应式移动 UI 的核心。Jetpack Compose 提供了懒加载布局，按需组合条目，而非一次性组合所有条目。

## LazyColumn 和 LazyRow

这些 composable 只组合可见条目，因此在大列表场景下比 `Column`/`Row` 更高效，后者会预先组合所有子项。

### LazyColumn（垂直滚动）
```kotlin
LazyColumn(modifier = Modifier.fillMaxSize()) {
  item {
    HeaderComposable()
  }
  items(itemList.size) { index ->
    ListItemComposable(itemList[index])
  }
  item {
    FooterComposable()
  }
}
```

### LazyRow（水平滚动）
```kotlin
LazyRow(modifier = Modifier.fillMaxWidth()) {
  items(imageList.size) { index ->
    Image(
      painter = painterResource(imageList[index]),
      contentDescription = null,
      modifier = Modifier.width(200.dp)
    )
  }
}
```

**与 Column/Row 的关键区别：** 条目在进入视口时才懒加载组合，减少内存和 CPU 使用。

**来源：** `androidx/compose/foundation/foundation/src/commonMain/kotlin/androidx/compose/foundation/lazy/`

## DSL 模式：item、items、itemsIndexed

### `item` — 单个 Composable
```kotlin
LazyColumn {
  item {
    HeaderComposable()
  }
}
```

### `items` — 来自 List 或 Count
```kotlin
// 来自 List
val users = listOf(User("Alice"), User("Bob"))
LazyColumn {
  items(users) { user ->
    UserCard(user)
  }
}

// 来自 count
LazyColumn {
  items(100) { index ->
    Text("Item $index")
  }
}
```

### `itemsIndexed` — 带索引
```kotlin
LazyColumn {
  itemsIndexed(users) { index, user ->
    Text("${index + 1}. ${user.name}")
  }
}
```

## Key：正确性和性能的关键

`key` 参数确保 Compose 能正确识别和复用条目，即使列表被重新排序。

### ✓ 良好：稳定的 Key
```kotlin
data class User(val id: Long, val name: String)

LazyColumn {
  items(users, key = { it.id }) { user ->
    UserCard(user)
  }
}
```

当 `users` 列表被重新排序时，Compose 知道哪个条目移动了，因为 key（id）是稳定的。

### ✗ 糟糕：使用索引作为 Key
```kotlin
// 避免：如果列表被重新排序，状态会混淆
LazyColumn {
  items(users, key = { index }) { user -> // 错误！
    var selected by remember { mutableStateOf(false) }
    UserCard(user, selected)
  }
}
```

如果你移除索引 0 处的条目，原来在索引 1 的条目移动到索引 0 并错误地继承了状态。

### ✗ 糟糕：无 Key
```kotlin
// 如果列表变化，条目状态/动画可能异常
LazyColumn {
  items(users) { user ->
    UserCard(user)
  }
}
```

没有 key，如果列表变化，Compose 无法可靠地区分条目。

**规则：** 当列表可能变化时，始终提供稳定、唯一的 key。使用 ID，而非索引。

## ContentType 用于复用优化

使用 `contentType` 在渲染不同类型条目时启用布局复用：

```kotlin
sealed class ListItem
data class HeaderItem(val title: String) : ListItem()
data class UserItem(val user: User) : ListItem()

LazyColumn {
  items(
    items = listItems,
    key = { it.id },
    contentType = { when (it) {
      is HeaderItem -> "header"
      is UserItem -> "user"
    }}
  ) { item ->
    when (item) {
      is HeaderItem -> HeaderComposable(item)
      is UserItem -> UserCard(item)
    }
  }
}
```

相同 `contentType` 的条目可以复用布局状态，在类型重复时提升性能。

## LazyListState：滚动位置和动画

以编程方式管理滚动位置：

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
  items(100) { index ->
    Text("Item $index")
  }
}

// 滚动到第 50 项
LaunchedEffect(Unit) {
  listState.scrollToItem(50)
}

// 带动画滚动
LaunchedEffect(Unit) {
  listState.animateScrollToItem(50)
}

// 读取当前滚动位置
val firstVisibleIndex = listState.firstVisibleItemIndex
val firstVisibleOffset = listState.firstVisibleItemScrollOffset
```

**使用场景：** 滚动到新添加的条目，或在用户操作时滚动。

## LazyVerticalGrid 和 LazyHorizontalGrid

### 固定列数
```kotlin
LazyVerticalGrid(columns = GridCells.Fixed(3)) {
  items(itemList.size) { index ->
    GridItemComposable(itemList[index])
  }
}
```

### 自适应列数（响应式）
```kotlin
// 列宽约 100dp，填充可用空间，尽可能容纳多列
LazyVerticalGrid(columns = GridCells.Adaptive(100.dp)) {
  items(itemList.size) { index ->
    GridItemComposable(itemList[index])
  }
}
```

自适应更适合响应式布局。

## LazyVerticalStaggeredGrid

用于 Pinterest 风格的可变高度布局：

```kotlin
LazyVerticalStaggeredGrid(
  columns = StaggeredGridCells.Fixed(2),
  modifier = Modifier.fillMaxSize()
) {
  items(images.size) { index ->
    AsyncImage(
      model = images[index].url,
      contentDescription = null,
      modifier = Modifier.fillMaxWidth()
    )
  }
}
```

条目流入当前高度最短的列，创建自然的交错外观。

## HorizontalPager 和 VerticalPager

逐页水平或垂直滑动：

```kotlin
val pagerState = rememberPagerState(pageCount = { pages.size })

HorizontalPager(state = pagerState) { page ->
  PageComposable(pages[page])
}

// 编程滚动到页面
LaunchedEffect(Unit) {
  pagerState.scrollToPage(2)
}

// 带动画滚动到页面
LaunchedEffect(Unit) {
  pagerState.animateScrollToPage(2)
}
```

## Lazy List 中的粘性头部

滚动时始终保持在顶部的头部：

```kotlin
LazyColumn {
  stickyHeader {
    SectionHeaderComposable("Section A")
  }
  items(itemsA) { item ->
    ItemComposable(item)
  }

  stickyHeader {
    SectionHeaderComposable("Section B")
  }
  items(itemsB) { item ->
    ItemComposable(item)
  }
}
```

## 嵌套滚动：陷阱

### ✗ 避免在 LazyColumn 内部使用可滚动组件
```kotlin
// 糟糕：嵌套滚动行为不可预测
LazyColumn {
  item {
    LazyRow { // 嵌套 lazy 可以，但...
      items(innerList) { item ->
        InnerItem(item)
      }
    }
  }
}
```

嵌套懒加载 composable 可以接受，但需要仔细考虑滚动优先级。

### ✗ 避免在 LazyColumn 内部使用 verticalScroll Modifier
```kotlin
// 糟糕：两个滚动容器争夺输入
LazyColumn {
  item {
    Column(modifier = Modifier.verticalScroll(rememberScrollState())) {
      Text("This is scrollable twice!")
    }
  }
}
```

不要在懒加载子项中包裹可滚动 modifier；如果需要多轴滚动，使用嵌套懒加载 composable。

### ✓ 复杂场景使用 nestedScroll
```kotlin
val scrollState = rememberScrollState()
val nestedScrollConnection = remember {
  object : NestedScrollConnection {
    override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
      // 自定义滚动处理
      return Offset.Zero
    }
  }
}

LazyColumn(modifier = Modifier.nestedScroll(nestedScrollConnection)) {
  items(100) { index ->
    Text("Item $index")
  }
}
```

## 性能：滚动依赖的 UI

### ✗ 糟糕：在 Item Lambda 中进行重度计算
```kotlin
LazyColumn {
  items(users) { user ->
    val processedData = expensiveComputation(user) // 每次重组都运行！
    UserCard(user, processedData)
  }
}
```

### ✓ 良好：使用 derivedStateOf 处理滚动依赖逻辑
```kotlin
val listState = rememberLazyListState()
val showScrollToTop = remember {
  derivedStateOf { listState.firstVisibleItemIndex > 0 }
}

LazyColumn(state = listState) {
  items(100) { index ->
    Text("Item $index")
  }
}

if (showScrollToTop.value) {
  Button(onClick = { /* scroll up */ }) { Text("Top") }
}
```

`derivedStateOf` 在滚动状态变化时派生新值，而不会重组整个列表。

## 反模式

### ✗ 对小规模固定列表使用 LazyColumn
```kotlin
// 糟糕：5 个条目不值得
LazyColumn {
  items(5) { index ->
    Text("Item $index")
  }
}
```

**更好：** 对小规模固定列表使用 `Column`。

### ✗ 无 Key + 列表变更
```kotlin
var items by remember { mutableStateOf(initialList) }
LazyColumn {
  items(items) { item -> // 无 key！
    ItemComposable(item, onDelete = {
      items = items.filter { it.id != item.id }
    })
  }
}
```

没有 key，移除条目会破坏剩余条目的状态。

### ✗ 在 Key 中创建新对象
```kotlin
// 糟糕：每次重组 key 都创建新对象
LazyColumn {
  items(users, key = { User(it.id, it.name) }) { user ->
    UserCard(user)
  }
}
```

**更好：** 使用原始类型稳定标识符。

## 关键要点

1. 对可变列表使用 `items` 时，始终提供稳定、唯一的 key
2. 对多类型列表使用 `contentType` 以启用布局复用
3. 对响应式网格布局优先使用 `GridCells.Adaptive`
4. 避免嵌套可滚动组件；对复杂滚动行为使用 `nestedScroll`
5. 使用 `derivedStateOf` 避免因滚动依赖逻辑而重组整个列表
6. `LazyColumn`/`LazyRow` 用于大规模或无界列表；小规模固定列表使用 `Column`/`Row`
7. 永远不要使用索引作为 key；列表变更会破坏条目状态

### 生产环境崩溃模式

#### indexOf() 在 items {} 内部 — O(n²) 且崩溃

```kotlin
// BAD: 总体 O(n²)， recreated 对象上返回 -1 → IndexOutOfBoundsException
items(list.size) { index ->
    val item = list[index]
    val position = list.indexOf(item) // 如果对象被 recreated 则为 -1！
}

// GOOD: 使用带 key 的 items() 获得稳定标识
items(list, key = { it.id }) { item ->
    ItemRow(item)
}
```

根本原因：`indexOf()` 使用 `equals()`。如果列表条目被重新创建（来自网络响应映射到数据类的新对象实例，没有正确的 `equals()` 实现），`indexOf()` 返回 -1。

#### 重复的 LazyColumn Key

后端发送无唯一 ID 的条目，或 WebSocket 重连在去重前发送重复消息 → `IllegalArgumentException: Key X was already used`。

```kotlin
// BAD: 后端 ID 可能不唯一
items(messages, key = { it.id }) { msg -> ... }

// GOOD: 为安全添加去重索引
items(messages, key = { "${it.id}_${it.dedupIndex}" }) { msg -> ... }
```

`dedupIndex` 模式：在数据类中添加一个字段，排除在 `equals()`/`hashCode()` 之外但包含在 LazyColumn key 中：

```kotlin
data class ChatMessage(
    val id: String,
    val text: String,
    val timestamp: Long
) {
    // 不在数据类构造函数中 — 排除在 equals/hashCode 之外
    var dedupIndex: Int = 0
}

// 处理后端消息时：
fun deduplicateKeys(messages: List<ChatMessage>): List<ChatMessage> {
    val seen = mutableMapOf<String, Int>()
    return messages.map { msg ->
        val count = seen.getOrPut(msg.id) { 0 }
        seen[msg.id] = count + 1
        msg.also { it.dedupIndex = count }
    }
}
```

#### derivedStateOf 驱动集合大小 → IOOB

```kotlin
// BAD: derived count 在 items{} 读取时可能已过期
val itemCount by remember { derivedStateOf { filterItems(allItems).size } }
LazyColumn {
    items(itemCount) { index ->
        val item = filterItems(allItems)[index] // 如果 allItems 变了则 IOOB！
    }
}

// GOOD: 派生完整过滤列表，而非仅 count
val filteredItems by remember { derivedStateOf { filterItems(allItems) } }
LazyColumn {
    items(filteredItems, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

规则：`derivedStateOf` 用于滚动方向、可见性、表单验证 — **永远不要用于驱动 LazyList 渲染的条目数量**。

### LazyList 加固

#### 带碰撞前缀的多字段 Key

当 LazyList 混合来自不同数据源的项目时，ID 可能冲突：

```kotlin
// BAD: liveItems 和 archivedItems 中都有 id=42 → 崩溃
LazyColumn {
    items(liveItems, key = { it.id }) { ... }
    items(archivedItems, key = { it.id }) { ... }
}

// GOOD: 按来源前缀 key
LazyColumn {
    items(liveItems, key = { "live_${it.id}" }) { ... }
    items(archivedItems, key = { "archived_${it.id}" }) { ... }
    items(pinnedItems, key = { "pinned_${it.id}" }) { ... }
}
```

#### 优先使用带 key 的 items() 而非 itemsIndexed()

`items(list, key = { it.id })` 为每个条目提供跨重组的稳定标识。这启用：
- 正确的 `animateItem()` 动画
- 高效的 diffing（只有变化的条目重组）
- 每个条目的状态正确保持

仅在真正需要索引用于显示时才使用 `itemsIndexed()`（例如编号列表）。

#### animateItem() 参数

```kotlin
items(items, key = { it.id }) { item ->
    ItemRow(
        item = item,
        modifier = Modifier.animateItem(
            fadeInSpec = tween(durationMillis = 250),
            placementSpec = spring(
                dampingRatio = Spring.DampingRatioLowBouncy,
                stiffness = Spring.StiffnessMediumLow
            ),
            fadeOutSpec = tween(durationMillis = 150)
        )
    )
}
```

#### ReportDrawnWhen 用于启动性能

向系统信号有意义的首屏内容已可见：

```kotlin
@Composable
fun ConversationListScreen(items: List<Conversation>) {
    ReportDrawnWhen { items.isNotEmpty() }

    LazyColumn {
        items(items, key = { it.id }) { item ->
            ConversationRow(item)
        }
    }
}
```

这改善了 Android vitals 中的 Time To Initial Display (TTID) 和 Time To Full Display (TTFD) 指标。

#### 设备特定的分页

某些设备（特别是三星）处理滚动事件不同，懒加载触发需要更少的滚动确认。实现无限滚动时，在多个 OEM 上测试并考虑可配置的滚动阈值。

```kotlin
val shouldLoadMore by remember {
    derivedStateOf {
        val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
        val totalItems = listState.layoutInfo.totalItemsCount
        lastVisibleItem >= totalItems - PREFETCH_THRESHOLD // 例如 5
    }
}

LaunchedEffect(shouldLoadMore) {
    if (shouldLoadMore) { viewModel.loadNextPage() }
}
```
