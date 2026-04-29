# Jetpack Compose 生产环境崩溃手册

大规模 Compose 应用中观察到的真实崩溃模式。每节记录根本原因、失败模式、修复方案和防止复发的规则。

---

## 1. remember {} 缺少配置派生 Key

### 根本原因

无 key 的 `remember {}` 缓存初始计算且永不重新计算。当记住的值派生自配置状态（屏幕尺寸、字体缩放、密度）时，配置变化（旋转、折叠姿态、窗口大小调整）会使缓存值过期。

### 崩溃模式

```kotlin
// BAD: shimmerCount 从初始 screenHeightDp 缓存，旋转后过期
@Composable
fun ShimmerList() {
    val config = LocalConfiguration.current
    val screenHeightDp = config.screenHeightDp
    val itemHeightDp = 80

    val shimmerCount = remember { (screenHeightDp / itemHeightDp).toInt() }

    LazyColumn {
        items(shimmerCount) { ShimmerItem() }
    }
}
```

旋转后，`screenHeightDp` 变化但 `shimmerCount` 仍持有竖屏值。列表渲染错误数量的 shimmer 占位符。极端情况下（折叠屏展开），过期数量导致布局溢出或零条目。

### 修复

```kotlin
// GOOD: screenHeightDp 作为 key，因此 remember 在配置变化时重新计算
@Composable
fun ShimmerList() {
    val config = LocalConfiguration.current
    val screenHeightDp = config.screenHeightDp
    val itemHeightDp = 80

    val shimmerCount = remember(screenHeightDp) {
        (screenHeightDp / itemHeightDp).toInt().coerceAtLeast(1)
    }

    LazyColumn {
        items(shimmerCount) { ShimmerItem() }
    }
}
```

**规则：** 任何派生自 `LocalConfiguration`、`LocalDensity` 或 `LocalLayoutDirection` 的值必须将该配置源包含在 `remember` 的 key 参数中。审计所有引用 `screenHeightDp`、`screenWidthDp`、`fontScale` 或 `densityDpi` 的 `remember {}` 调用。

---

## 2. indexOf() 在 items {} 内部

### 根本原因

在 LazyColumn 的 `items {}` 块内部使用 `list.indexOf(item)` 是每项 O(n)，使整个列表变为 O(n²)。更糟的是，`indexOf` 使用结构相等性（`equals`）。如果列表包含重新创建的对象（来自网络响应映射到数据类的新实例），`indexOf` 可能返回 `-1`，作为索引传递时会触发 `IndexOutOfBoundsException`。

### 崩溃模式

```kotlin
// BAD: O(n²) 且 indexOf 返回 -1 时崩溃
@Composable
fun MessageList(messages: List<Message>) {
    LazyColumn {
        items(messages) { message ->
            val index = messages.indexOf(message)  // 每项 O(n)
            MessageRow(
                message = message,
                isEven = index % 2 == 0  // -1 % 2 == -1，此处不会崩溃
            )
        }
    }
}

// 更糟：将 indexOf 结果作为直接索引使用
items(messages) { message ->
    val index = messages.indexOf(message)
    val nextMessage = messages[index + 1]  // IndexOutOfBoundsException
}
```

### 修复

```kotlin
// GOOD: 使用 itemsIndexed 直接获取索引
@Composable
fun MessageList(messages: List<Message>) {
    LazyColumn {
        itemsIndexed(
            items = messages,
            key = { _, message -> message.id }
        ) { index, message ->
            MessageRow(
                message = message,
                isEven = index % 2 == 0
            )
        }
    }
}
```

如果你需要带 key 的 `items` 但也需要索引，尽可能从条目自身推导索引，或切换到 `itemsIndexed`。

**规则：** 绝不要在 `LazyListScope` 条目工厂内部调用 `indexOf()`、`lastIndexOf()` 或 `indexOfFirst {}`。使用 `itemsIndexed` 访问索引。使用 `items(key = { ... })` 获得稳定标识。

---

## 3. DrawScope 缺少零尺寸防护

### 根本原因

在初始组合期间，`Canvas` composable 可能在布局完成前接收到 `Size.Zero`（或 `Size.Unspecified`），尤其是在父组件尚未测量时使用 `fillMaxSize()`。除以零维度值或在 `size.minDimension` 为零时计算半径会产生 `NaN` 或 `Infinity`，从而崩溃 Skia 渲染管线。

### 崩溃模式

```kotlin
// BAD: 初始组合期间 size.minDimension 为 0
Canvas(modifier = Modifier.fillMaxSize()) {
    val radius = size.minDimension / 2
    drawCircle(color = Color.Blue, radius = radius)
}
```

在某些设备上或 `SubcomposeLayout` 内部，第一次绘制调用以 `size = Size(0, 0)` 触发。`radius = 0f` 的 `drawCircle` 调用可能不会崩溃，但派生数学如 `360f / size.width` 会产生 `Infinity` 并损坏绘制路径。

### 修复

```kotlin
// GOOD: 防护零尺寸，并给 Canvas 显式尺寸
Canvas(
    modifier = Modifier
        .fillMaxWidth()
        .height(200.dp)
) {
    if (size.minDimension <= 0f) return@Canvas
    val radius = size.minDimension / 2
    drawCircle(color = Color.Blue, radius = radius)
}
```

对于动态尺寸，使用 `Modifier.aspectRatio()` 或 `BoxWithConstraints` 保证绘制前维度非零：

```kotlin
BoxWithConstraints(modifier = Modifier.fillMaxWidth()) {
    if (maxWidth > 0.dp && maxHeight > 0.dp) {
        Canvas(modifier = Modifier.size(maxWidth, maxWidth)) {
            val radius = size.minDimension / 2
            drawCircle(color = Color.Blue, radius = radius)
        }
    }
}
```

**规则：** 始终防护 `DrawScope` 块防止零尺寸条件。永远不要在没有显式高度约束的情况下对 `Canvas` 使用 `fillMaxSize()`。优先使用 `Modifier.size()`、`Modifier.height()` 或 `Modifier.aspectRatio()` 使 Canvas 始终以已知维度进入绘制。

---

## 4. 重复的 LazyColumn Key

### 根本原因

`LazyColumn` 要求所有条目的 key 唯一。当后端发送无唯一 ID 的条目，或 WebSocket 重连在去重前发送重复消息时，重复 key 导致 `IllegalArgumentException: Key ... was already used`。

### 崩溃模式

```kotlin
// BAD: WebSocket 重连后后端发送重复 ID
data class Notification(val id: String, val text: String)

LazyColumn {
    items(
        items = notifications,
        key = { it.id }  // 如果两个条目共享相同 id 则崩溃
    ) { notification ->
        NotificationRow(notification)
    }
}
```

### 修复：去重索引模式

添加一个排除在 `equals`/`hashCode` 之外但包含在 key 中的 `dedupIndex` 字段。这能在不丢失数据的情况下优雅处理重复：

```kotlin
data class Notification(
    val id: String,
    val text: String,
    val timestamp: Long
) {
    // dedupIndex 不在主构造函数中，排除在 equals/hashCode 之外
    var dedupIndex: Int = 0
}

fun List<Notification>.withDedupIndex(): List<Notification> {
    val seen = mutableMapOf<String, Int>()
    return map { item ->
        val count = seen.getOrDefault(item.id, 0)
        seen[item.id] = count + 1
        item.also { it.dedupIndex = count }
    }
}

@Composable
fun NotificationList(notifications: List<Notification>) {
    val deduped = remember(notifications) { notifications.withDedupIndex() }

    LazyColumn {
        items(
            items = deduped,
            key = { "${it.id}_${it.dedupIndex}" }  // 保证唯一
        ) { notification ->
            NotificationRow(notification)
        }
    }
}
```

当应丢弃真正重复项时，使用 `distinctBy` 的替代防御方法：

```kotlin
val uniqueNotifications = remember(notifications) {
    notifications.distinctBy { it.id }
}
```

**规则：** 永远不要信任后端数据提供唯一 key。要么用 `distinctBy` 去重，要么使用去重索引模式使 key 唯一。将 key 构造包装在可单元测试的函数中。

---

## 5. derivedStateOf 驱动集合大小

### 根本原因

`derivedStateOf` 在输入变化时惰性重新计算，但读取派生值的重组和读取源集合的重组可能发生在不同帧。当 `derivedStateOf` 只暴露 count 时，`items(count)` 调用可能使用与 actual 集合不同步的过期 count，导致 `IndexOutOfBoundsException`。

### 崩溃模式

```kotlin
// BAD: 只派生 count；items {} 直接用 allItems 读取，count 可能已过期
@Composable
fun FilteredList(allItems: List<Item>, filter: String) {
    val itemCount by remember {
        derivedStateOf { allItems.count { it.name.contains(filter) } }
    }

    LazyColumn {
        items(itemCount) { index ->
            val item = allItems.filter { it.name.contains(filter) }[index]
            // itemCount 可能相对于 allItems 已过期，IOOB 崩溃
            ItemRow(item)
        }
    }
}
```

### 修复

```kotlin
// GOOD: 派生完整过滤列表；count 和访问始终一致
@Composable
fun FilteredList(allItems: List<Item>, filter: String) {
    val filteredItems by remember {
        derivedStateOf { allItems.filter { it.name.contains(filter) } }
    }

    LazyColumn {
        items(
            items = filteredItems,
            key = { it.id }
        ) { item ->
            ItemRow(item)
        }
    }
}
```

**规则：** `derivedStateOf` 适用于滚动方向、可见性阈值和验证状态 — 不驱动集合迭代的值。永远不要使用 `derivedStateOf` 暴露 LazyList 将用来访问单独集合的 count 或索引。

---

## 6. collectAsState 与 collectAsStateWithLifecycle

### 根本原因

`collectAsState()` 订阅 `Flow` 并持续收集，即使应用在后台。这导致不必要的工作（网络调用、数据库查询、传感器读取）、电池消耗，以及用户返回应用时短暂闪烁的过期状态。

### 区别

```kotlin
// BAD: 在后台继续收集
@Composable
fun ProfileScreen(viewModel: ProfileViewModel) {
    val state by viewModel.uiState.collectAsState()
    ProfileContent(state)
}

// GOOD: 生命周期低于 STARTED 时停止收集
@Composable
fun ProfileScreen(viewModel: ProfileViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    ProfileContent(state)
}
```

`collectAsStateWithLifecycle` 属于 `androidx.lifecycle:lifecycle-runtime-compose`。当 `Lifecycle` 低于可配置状态（默认：`Lifecycle.State.STARTED`）时自动取消收集。

### 自定义生命周期最小值

```kotlin
// 仅在 RESUMED 时收集（更严格，用于相机/位置 flow）
val locationState by locationFlow.collectAsStateWithLifecycle(
    minActiveState = Lifecycle.State.RESUMED
)
```

**规则：** 在 Android 上，始终对 `StateFlow` 和 `SharedFlow` 使用 `collectAsStateWithLifecycle`。仅对无生命周期感知的上下文保留 `collectAsState`。

---

## 7. SafeShimmerItem 模式

### 根本原因

通过 `SubcomposeLayout` 渲染的 shimmer 占位符可能在父布局尺寸为零时崩溃（见第 3 节），或当 shimmer 动画的 `DrawScope` 接收到意外维度时崩溃。防御性包装器防止崩溃同时保持 shimmer UX。

### 完整实现

```kotlin
@Composable
fun SafeShimmerItem(
    modifier: Modifier = Modifier,
    itemHeight: Dp = 80.dp,
    shimmerColor: Color = Color.LightGray.copy(alpha = 0.6f),
    highlightColor: Color = Color.LightGray.copy(alpha = 0.2f),
) {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = 0f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer_translate"
    )

    val brush = Brush.linearGradient(
        colors = listOf(shimmerColor, highlightColor, shimmerColor),
        start = Offset(translateAnim - 500f, 0f),
        end = Offset(translateAnim, 0f)
    )

    Box(
        modifier = modifier
            .fillMaxWidth()
            .height(itemHeight)
            .clip(RoundedCornerShape(8.dp))
    ) {
        val result = runCatching {
            Canvas(modifier = Modifier.matchParentSize()) {
                if (size.minDimension <= 0f) return@Canvas
                drawRect(brush = brush)
            }
        }
        if (result.isFailure) {
            // 降级：Canvas 失败时使用纯色占位符
            Box(
                modifier = Modifier
                    .matchParentSize()
                    .background(shimmerColor, RoundedCornerShape(8.dp))
            )
        }
    }
}
```

### 与配置感知数量一起使用

```kotlin
@Composable
fun ShimmerLoadingList() {
    val config = LocalConfiguration.current
    val itemHeight = 80.dp
    val shimmerCount = remember(config.screenHeightDp) {
        (config.screenHeightDp / itemHeight.value).toInt().coerceAtLeast(1)
    }

    LazyColumn {
        items(shimmerCount) {
            SafeShimmerItem(itemHeight = itemHeight)
        }
    }
}
```

**规则：** 将所有 shimmer 和占位符绘制包装在 `runCatching` 或显式尺寸防护中。提供纯色降级，使加载状态永远不会不可见，也永远不会崩溃。

---

## 8. 带碰撞前缀的多字段 Key

### 根本原因

当 `LazyColumn` 显示来自多个类别（实时、已归档、置顶）的条目时，对同一底层实体使用原始 ID 作为 key 会在多个 section 中导致碰撞。例如，一条 `id = 42` 的消息可能同时出现在 "pinned" 和 "live" section 中。

### 崩溃模式

```kotlin
// BAD: pinned 和 live section 中相同消息 ID 导致 key 碰撞
LazyColumn {
    items(pinnedMessages, key = { it.id }) { message ->
        PinnedMessageRow(message)
    }
    items(liveMessages, key = { it.id }) { message ->
        LiveMessageRow(message)
    }
}
// IllegalArgumentException: Key 42 was already used
```

### 修复：类型前缀 Key

```kotlin
// GOOD: 用 section 类型前缀 key 避免碰撞
LazyColumn {
    items(
        items = pinnedMessages,
        key = { "pinned_${it.id}" }
    ) { message ->
        PinnedMessageRow(message)
    }
    items(
        items = liveMessages,
        key = { "live_${it.id}" }
    ) { message ->
        LiveMessageRow(message)
    }
    items(
        items = archivedMessages,
        key = { "archived_${it.id}" }
    ) { message ->
        ArchivedMessageRow(message)
    }
}
```

### 密封类替代方案

为了类型安全，对 key 使用密封类：

```kotlin
sealed class ListKey {
    data class Pinned(val id: Long) : ListKey()
    data class Live(val id: Long) : ListKey()
    data class Archived(val id: Long) : ListKey()
}

LazyColumn {
    items(pinnedMessages, key = { ListKey.Pinned(it.id) }) { ... }
    items(liveMessages, key = { ListKey.Live(it.id) }) { ... }
    items(archivedMessages, key = { ListKey.Archived(it.id) }) { ... }
}
```

**规则：** 在单个 `LazyList` 中混合条目类型时，始终用条目类型前缀或包装 key。字符串前缀简单，密封类提供编译时安全。

---

## 9. 生产环境状态规则

这些规则防止生产环境 Compose 应用中最常见的状态相关崩溃和架构错误。

### 规则 1：mutableStateOf 仅在 Composable 中使用，绝不在 ViewModel 中

```kotlin
// BAD: ViewModel 中的 Compose 状态将 VM 与 Compose 运行时耦合
class ProfileViewModel : ViewModel() {
    var name by mutableStateOf("")  // 不要这样做
}

// GOOD: ViewModel 中使用协程原生状态
class ProfileViewModel : ViewModel() {
    private val _name = MutableStateFlow("")
    val name: StateFlow<String> = _name.asStateFlow()
}
```

ViewModel 应暴露 `StateFlow`（通过 `MutableStateFlow` + `asStateFlow()`）。Compose 状态（`mutableStateOf`）属于 `@Composable` 函数和标注 `@Stable` 的状态持有者类。这使 ViewModel 无需 Compose 运行时即可测试。

### 规则 2：事件使用 SharedFlow，而非 Channel

```kotlin
// BAD: Channel 在无活跃收集器时丢弃事件
class OrderViewModel : ViewModel() {
    private val _events = Channel<UiEvent>()
    val events = _events.receiveAsFlow()
}

// GOOD: 带缓冲的 SharedFlow 处理短暂收集器间隙
class OrderViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>(
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()
}
```

`Channel` 是需要活跃收集器的热流。在配置变化或生命周期过渡期间，事件被静默丢弃。`extraBufferCapacity = 1` 的 `SharedFlow` 在短暂收集器间隙中缓冲一个事件。

### 规则 3：rememberSaveable 仅在 NavGraph 层级

```kotlin
// BAD: 列表项深处使用 rememberSaveable（膨胀 saved state bundle）
@Composable
fun ChatMessageItem(message: Message) {
    var expanded by rememberSaveable { mutableStateOf(false) }
    // 为列表中每条消息都保存 -- Bundle 大小爆炸
}

// GOOD: screen 级别使用 rememberSaveable，item 内部使用 remember
@Composable
fun ChatScreen(viewModel: ChatViewModel) {
    var searchQuery by rememberSaveable { mutableStateOf("") }

    LazyColumn {
        items(messages, key = { it.id }) { message ->
            var expanded by remember { mutableStateOf(false) }
            ChatMessageItem(message, expanded)
        }
    }
}
```

`rememberSaveable` 序列化到 `Bundle`，在 Android 上有 ~1MB 限制。在列表项内部用于每条目状态时，很快超过此限制并导致 `TransactionTooLargeException`。

### 规则 4：snapshotFlow + distinctUntilChanged 用于响应式滚动

```kotlin
// GOOD: 高效的响应式滚动位置监控
@Composable
fun ScrollAwareList(listState: LazyListState) {
    val isScrolledPast = remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    // 用于基于滚动位置的副作用
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                // Analytics、FAB 可见性、header 折叠
            }
    }
}
```

`snapshotFlow` 将 Compose 快照状态转换为 `Flow`，`distinctUntilChanged()` 防止冗余发射。绝不要在重组驱动的循环中轮询滚动状态。

### 规则 5：stateIn() 配合 map() 用于派生 flow

```kotlin
// GOOD: 从仓库 flow 派生 UI 状态
class DashboardViewModel(repository: DashboardRepository) : ViewModel() {
    val uiState: StateFlow<DashboardUiState> = repository.dashboardData
        .map { data ->
            DashboardUiState(
                totalSales = data.sales.sumOf { it.amount },
                topProduct = data.products.maxByOrNull { it.revenue }?.name.orEmpty(),
                isLoading = false
            )
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = DashboardUiState()
        )
}
```

`SharingStarted.WhileSubscribed(5_000)` 在最后一个订阅者断开连接后保持上游活跃 5 秒，在配置变化期间无需重启 flow 即可存活。配合 `.map()` 进行派生转换，而不是在 UI 中创建单独的 `derivedStateOf`。

---

## 10. 生产环境性能规则

### 规则 1：UI 状态使用 @Stable，数据模型使用 @Immutable

```kotlin
// 来自网络/数据库的数据模型：真正不可变
@Immutable
data class Product(
    val id: Long,
    val name: String,
    val price: Double,
    val imageUrl: String
)

// 有可观察突变的 UI 状态：稳定契约
@Stable
data class CartUiState(
    val items: List<Product> = emptyList(),
    val total: Double = 0.0,
    val isLoading: Boolean = false
)
```

`@Immutable` 告诉编译器所有属性在构造后永不改变。`@Stable` 承诺变化将通知 Compose 运行时。误用（例如在带有 `var` 属性的类上使用 `@Immutable`）会导致跳过重组和过期 UI。

### 规则 2：Canvas 始终显式定尺寸

```kotlin
// BAD: Canvas 带 fillMaxSize 且无父约束
Canvas(modifier = Modifier.fillMaxSize()) { /* 可能获得零尺寸 */ }

// GOOD: 显式维度
Canvas(modifier = Modifier.size(200.dp)) { ... }

// GOOD: 显式高度配灵活宽度
Canvas(modifier = Modifier.fillMaxWidth().height(120.dp)) { ... }

// GOOD: 宽高比
Canvas(modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)) { ... }
```

参见第 3 节了解详细崩溃场景。显式尺寸防止零尺寸绘制调用。

### 规则 3：ReportDrawnWhen 用于启动信号

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    ReportDrawnWhen { state.isContentReady }

    when {
        state.isLoading -> ShimmerLoadingList()
        state.isContentReady -> ContentList(state.items)
    }
}
```

`ReportDrawnWhen` 向系统（和性能监控工具）信号屏幕已有有意义内容。这对准确的 Time-To-Initial-Display (TTID) 和 Time-To-Full-Display (TTFD) 指标至关重要。

### 规则 4：R8 在 Release 中剥离 Preview 和 Semantics

```kotlin
// 在 build.gradle.kts（app 模块）
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

R8 自动从 release 构建中剥离 `@Preview` composable。

### 规则 5：务实的集合稳定性

```kotlin
// 当 composable 接收一个你知道不会突变的 List 时：
@Suppress("ComposeUnstableCollections")
@Composable
fun ProductGrid(
    products: List<Product>,  // 编译器将 List 视为不稳定
    onProductClick: (Product) -> Unit
) {
    LazyVerticalGrid(columns = GridCells.Fixed(2)) {
        items(products, key = { it.id }) { product ->
            ProductCard(product, onProductClick)
        }
    }
}
```

Compose 编译器将 `List`、`Map`、`Set` 视为不稳定，因为它们可能是可变实现支持的接口。处理选项：

1. **`@Suppress("ComposeUnstableCollections")`** -- 务实，每函数 opt-out
2. **包装在 `@Immutable` 持有者中** -- `@Immutable data class ProductList(val items: List<Product>)`
3. **使用 `kotlinx.collections.immutable`** -- `ImmutableList<Product>` 被识别为稳定
4. **Compose 编译器稳定性配置文件** -- 全局声明稳定类

基于团队约定选择。选项 1 对现有代码库最快。选项 3 对新项目最干净。

---

## 速查：崩溃模式检查清单

| 模式 | 症状 | 章节 |
|---------|---------|---------|
| 无配置 key 的 `remember {}` | 旋转后值过期 | 1 |
| `items {}` 中的 `indexOf()` | O(n²)、IndexOutOfBoundsException | 2 |
| 零尺寸 Canvas | draw 中 NaN/Infinity、Skia 崩溃 | 3 |
| 重复的 LazyColumn key | IllegalArgumentException | 4 |
| 集合 count 的 `derivedStateOf` | IndexOutOfBoundsException | 5 |
| 后台中的 `collectAsState` | 电池消耗、过期闪烁 | 6 |
| 无防护的 shimmer 绘制 | SubcomposeLayout 中崩溃 | 7 |
| 跨条目类型的 key 碰撞 | IllegalArgumentException | 8 |
| ViewModel 中的 `mutableStateOf` | 不可测试、生命周期不匹配 | 9 |
| 仅 `fillMaxSize()` 的 Canvas | 零尺寸绘制崩溃 | 10 |
