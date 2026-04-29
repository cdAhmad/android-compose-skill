# Jetpack Compose 状态管理参考

## 状态基础

Compose 中的状态是可观察数据，变化时会触发重组。

### 创建状态

使用类型特定的状态持有者以提升效率：

```kotlin
// 通用状态（任意类型）
val name = mutableStateOf("Alice")

// 原始类型特化（避免装箱）
val count = mutableIntStateOf(0)
val progress = mutableFloatStateOf(0.5f)
val enabled = mutableStateOf(true)  // Boolean 无特化版
```

**陷阱：** 使用 `mutableStateOf<Int>()` 而非 `mutableIntStateOf()` 会在每次读/写时产生不必要的装箱。原始类型特化函数位于 `androidx.compose.runtime`（来源：`State.kt`）。

## remember 与 rememberSaveable

两者都将状态与组合 key 关联，但持久化范围不同。

### remember
- 存活于组合的生命周期内
- 进程死亡、配置变化、返回导航时丢失
- 最适合 UI 状态：选中、展开/折叠、滚动位置

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

### rememberSaveable
- 在进程死亡和配置变化中存活
- 默认使用 `Bundle` 兼容类型（String、Int、Boolean 等）
- 对于自定义类型，提供 `Saver` 或使用 `@Parcelize`
- 最适合代表用户输入或导航状态的数据

```kotlin
@Composable
fun SearchScreen() {
    var query by rememberSaveable { mutableStateOf("") }
    // 在配置变化中存活
}

// 自定义类型需要显式 Saver
data class User(val id: Int, val name: String)
val userSaver = Saver<User, String>(
    save = { "${it.id}:${it.name}" },
    restore = { parts -> User(parts.split(":")[0].toInt(), parts.split(":")[1]) }
)
var user by rememberSaveable(stateSaver = userSaver) { mutableStateOf(User(1, "Alice")) }
```

**陷阱：** 假设 `rememberSaveable` 适用于所有类型。自定义类需要显式 `Saver` 或 `@Parcelize`。参见 `SaveableStateRegistry` 在 `androidx.compose.runtime.saveable`。

## 状态提升

将状态移到父 composable 中以实现可复用性和可测试性。

### 有状态 vs 无状态模式

```kotlin
// ❌ 有状态版本（紧耦合）
@Composable
fun Counter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) { Text(count.toString()) }
}

// ✅ 无状态版本（可复用、可测试）
@Composable
fun Counter(
    count: Int,
    onCountChange: (Int) -> Unit
) {
    Button(onClick = { onCountChange(count + 1) }) { Text(count.toString()) }
}

// ✅ 包裹 composable（提供状态，使用无状态子组件）
@Composable
fun StatefulCounter() {
    var count by remember { mutableIntStateOf(0) }
    Counter(count = count, onCountChange = { count = it })
}
```

**规则：** 按需将状态推到最高层，但不要更高。如果只有一个子组件需要状态，就放在那里。如果多个子组件或父组件需要，就向上提升。

## derivedStateOf

从现有状态计算值，仅在依赖变化时重新计算。

```kotlin
// ❌ 错误：每次重组都重新计算
val isEven = count % 2 == 0

// ✅ 正确：仅在 count 变化时重新计算
val isEven = derivedStateOf { count % 2 == 0 }
```

**何时使用：**
- 来自状态的昂贵计算（例如过滤、排序列表）
- 组合多个状态值
- 为条件逻辑创建中间状态

```kotlin
@Composable
fun UserList(users: List<User>, filterText: String) {
    val filteredUsers = derivedStateOf {
        users.filter { it.name.contains(filterText, ignoreCase = true) }
    }

    LazyColumn {
        items(filteredUsers.value.size) { index ->
            UserRow(filteredUsers.value[index])
        }
    }
}
```

**陷阱：** 对廉价操作（字符串拼接、简单条件）使用 `derivedStateOf` 会增加开销。仅在计算非平凡时使用。

**陷阱：** 在传递给子 composable 的 lambda 中访问 `.value` 不会创建依赖。对回调使用 `snapshotFlow`。

## snapshotFlow

将 Compose 状态转换为 Kotlin Flow 用于副作用和外部 API。

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by remember { mutableStateOf("") }

    LaunchedEffect(Unit) {
        snapshotFlow { query }
            .debounce(500)
            .distinctUntilChanged()
            .collect { viewModel.search(it) }
    }
}
```

**关键行为：**
- 发射初始值，然后仅在变化时发射
- 与 derivedStateOf、集合和嵌套状态一起工作
- 在组合的协程作用域中运行（通过 `LaunchedEffect` 启动）

**陷阱：** 在 `LaunchedEffect` 中直接访问状态不会跟踪变化：
```kotlin
// ❌ query 变化时不会重新运行
LaunchedEffect(Unit) {
    viewModel.search(query)  // 仅在启动时捕获
}

// ✅ query 变化时重新运行
LaunchedEffect(query) {
    viewModel.search(query)
}
```

## SnapshotStateList 和 SnapshotStateMap

在结构变化时触发重组的可观察集合。

```kotlin
val items = remember { mutableStateListOf<Item>() }
items.add(Item(1, "First"))
items[0] = Item(1, "Updated")
items.removeAt(0)

val map = remember { mutableStateMapOf<String, String>() }
map["key"] = "value"  // 触发重组
```

**重要：** 列表内容的变化会触发重组，但列表*元素*的变化（如果它们是可变对象）不会。

```kotlin
data class Item(val id: Int, var name: String)

val items = remember { mutableStateListOf(Item(1, "First")) }

// ✅ 触发重组（列表结构变化）
items[0] = Item(1, "Updated")

// ❌ 不会触发重组（对象原地突变）
items[0].name = "Updated"  // 已突变但列表引用未变

// ✅ 正确：使用 copy() 或 mutableStateOf 处理嵌套状态
items[0] = items[0].copy(name = "Updated")
```

参见来源：`androidx.compose.runtime.snapshots` 中的集合实现。

## @Stable 和 @Immutable 注解

这些注解帮助编译器优化重组（强跳过模式）。

### @Immutable
- 所有 public 字段是只读原始类型或其他 `@Immutable` 类型
- 实例在构造后永不改变
- 如果参数未变，编译器可以跳过重组

```kotlin
@Immutable
data class User(val id: Int, val name: String)
```

### @Stable
- 实现结构相等性（`equals`）
- public 属性是只读或可观察的
- 变化始终通知 Compose（通过状态对象）
- 比 `@Immutable` 保证更弱，但适合具有可观察状态的类型

```kotlin
@Stable
class UserViewModel {
    val userName: State<String> = mutableStateOf("")
    val isLoading: State<Boolean> = mutableStateOf(false)

    // 可观察状态，非直接属性
}
```

**陷阱：** 未注解用作参数的数据类。未注解类型被假定为不稳定，触发不必要的重组。

```kotlin
// ❌ 被视为不稳定，导致重组
class Config(val title: String, val color: Color)

// ✅ 正确注解
@Immutable
class Config(val title: String, val color: Color)
```

## 强跳过模式

在 Compose 1.6+ 中，强跳过模式应用更严格的重组逻辑。

**变化：**
- 如果*所有*参数的身份和值未变，composable 跳过重组
- 未注解的参数类型被视为不稳定（始终重组）
- `@Stable` 和 `@Immutable` 注解现在对性能至关重要
- Lambda 参数始终导致重组（它们是新实例）

**启用强跳过：**
```gradle
composeOptions {
    kotlinCompilerExtensionVersion = "1.5.4+"  // 默认启用
}
```

**实际影响：**
```kotlin
// ❌ 这些创建新实例，始终重组子组件
@Composable
fun Parent() {
    Child(title = buildString { append("Title") })
    Child(config = Config(...))  // 不稳定类型
}

// ✅ 缓存实例
@Composable
fun Parent() {
    val title = remember { "Title" }
    val config = remember { Config(...) }
    Child(title = title)
    Child(config = config)
}
```

## ViewModel 中的状态：StateFlow vs Compose 状态

### StateFlow（ViewModel 推荐）
- 在组合重组和配置变化中存活
- 与生命周期一起工作（`collectAsStateWithLifecycle`）
- 线程安全，跨层工作

```kotlin
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> SuccessScreen((uiState as UiState.Success).data)
        is UiState.Error -> ErrorScreen((uiState as UiState.Error).message)
    }
}
```

### Compose 状态（仅用于 UI 状态）
- 用于临时的、UI 本地状态
- 不要提升到 ViewModel
- 返回导航时丢失

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var showFilters by remember { mutableStateOf(false) }  // 仅 UI
    val searchResults by viewModel.searchResults.collectAsStateWithLifecycle()

    SearchUI(
        results = searchResults,
        showFilters = showFilters,
        onToggleFilters = { showFilters = !showFilters }
    )
}
```

**关键区别：** `collectAsStateWithLifecycle()`（在 `androidx.lifecycle:lifecycle-runtime-compose` 中）仅在 composable 处于 STARTED 状态时收集，避免内存泄漏。

## 常见反模式

### 本地变量中的状态
```kotlin
// ❌ 重组时丢失
@Composable
fun Counter() {
    var count = 0  // 每次重组都重置为 0
    Button(onClick = { count++ }) { Text(count.toString()) }
}

// ✅ 正确
@Composable
fun Counter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) { Text(count.toString()) }
}
```

### 在错误作用域中读取状态
```kotlin
// ❌ 在 lambda 内部读取；变化不会重新启动 effect
var count by remember { mutableIntStateOf(0) }
LaunchedEffect(Unit) {
    while (true) {
        delay(1000)
        println(count)  // 始终打印 0
    }
}

// ✅ 将状态传递给 LaunchedEffect key
LaunchedEffect(count) {
    println("Count changed: $count")
}
```

### 在 Lambda 中创建状态
```kotlin
// ❌ 每次调用都创建新状态
val onButtonClick = {
    val newValue = remember { mutableStateOf(0) }  // 错误：不能在 lambda 中调用 remember
}

// ✅ 在组合层级创建状态
var value by remember { mutableIntStateOf(0) }
val onButtonClick = { value++ }
```

---

**来源参考：** `androidx.compose.runtime.State`、`androidx.compose.runtime.saveable`、`androidx.lifecycle.runtime.compose`

---

## produceState

桥接挂起函数和 Compose 状态：

```kotlin
@Composable
fun UserProfile(userId: String): State<User?> = produceState<User?>(initialValue = null, userId) {
    value = repository.getUser(userId)
}
```

在需要将挂起函数结果转换为可观察状态时使用。协程作用域于组合，并在 composable 离开时取消。

也可以观察 flow：
```kotlin
@Composable
fun NetworkStatus(): State<Boolean> = produceState(initialValue = false) {
    connectivityManager.observeNetworkState().collect { value = it }
}
```

---

## rememberUpdatedState

在长时间运行的副作用中捕获最新 callback 值：

```kotlin
@Composable
fun Timer(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    LaunchedEffect(true) {
        delay(5000L)
        currentOnTimeout() // 始终调用最新的 onTimeout，即使它已变化
    }
}
```

使用场景：LaunchedEffect 捕获了可能变化的 callback，但你不想重启 effect。没有 `rememberUpdatedState`，effect 会使用过期的原始 callback 或需要在每次 callback 变化时重启。

---

## 密封 UiState 模式

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}
```

智能转换安全性：
```kotlin
// BAD: smart cast 可能在检查和用法之间失败
if (uiState is UiState.Success) {
    Content((uiState as UiState.Success).data) // 不安全转换
}

// GOOD: 用 val 捕获实现安全 smart cast
when (val state = uiState) {
    is UiState.Loading -> LoadingIndicator()
    is UiState.Success -> Content(state.data) // 通过 val 安全 smart cast
    is UiState.Error -> ErrorMessage(state.message)
}
```

---

## 状态持有者类模式

对于具有多个相互关联状态值的复杂屏幕，创建状态持有者：

```kotlin
@Composable
fun rememberSearchState(
    listState: LazyListState = rememberLazyListState(),
    coroutineScope: CoroutineScope = rememberCoroutineScope()
): SearchState = remember(listState, coroutineScope) {
    SearchState(listState, coroutineScope)
}

@Stable
class SearchState(
    val listState: LazyListState,
    private val coroutineScope: CoroutineScope
) {
    var query by mutableStateOf("")
        private set

    val isScrolled: Boolean
        get() = listState.firstVisibleItemIndex > 0

    fun updateQuery(newQuery: String) { query = newQuery }
    fun scrollToTop() { coroutineScope.launch { listState.animateScrollToItem(0) } }
}
```

此模式（由 `rememberScrollState`、`rememberDrawerState` 等使用）将相关状态和逻辑分组到单个类中，避免 composable 中的参数膨胀。

---

## 生产环境状态规则

### 1. mutableStateOf 仅在 composable 中使用，绝不在 ViewModel 中

```kotlin
// BAD: ViewModel 中的 Compose 状态将 VM 与 Compose 运行时耦合
class MyViewModel : ViewModel() {
    var name by mutableStateOf("") // 不要这样做
}

// GOOD: ViewModel 中使用 StateFlow — 与框架无关，可测试
class MyViewModel : ViewModel() {
    private val _name = MutableStateFlow("")
    val name = _name.asStateFlow()

    fun updateName(new: String) { _name.value = new }
}
```

### 2. 一次性事件使用 SharedFlow，而非 Channel

```kotlin
// GOOD: 带缓冲的 SharedFlow 处理一次性事件
private val _events = MutableSharedFlow<AppEvent>(extraBufferCapacity = 1)
val events = _events.asSharedFlow()

// 从 ViewModel 发射
fun onAction() { _events.tryEmit(AppEvent.ShowSnackbar("Done")) }

// 在 composable 中收集
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is AppEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
            is AppEvent.Navigate -> onNavigate(event.route)
        }
    }
}
```

### 3. rememberSaveable 仅在 NavGraph 层级

对 NavGraph 入口点的屏幕级状态（搜索查询、标签选择）使用 `rememberSaveable`，而不是在 composable 树深处使用，那样会增加不必要的持久化开销。

### 4. snapshotFlow + distinctUntilChanged() 用于响应式滚动

```kotlin
LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .distinctUntilChanged()
        .collect { index -> viewModel.onScrollPositionChanged(index) }
}
```

### 5. .stateIn() 配合 .map() 用于派生 flow

```kotlin
val filteredItems = repository.items
    .map { items -> items.filter { it.isActive } }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
```
