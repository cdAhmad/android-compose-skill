# Jetpack Compose 已弃用模式与 API 迁移

本指南涵盖 Compose 演进过程中的主要 API 变更和弃用。每节展示旧模式 → 新方法及迁移说明。

---

## 字符串路由 → 类型安全 `@Serializable` 路由

**旧（pre-2.8）：**
```kotlin
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen() }
    composable("details/{id}") { backStackEntry ->
        DetailsScreen(id = backStackEntry.arguments?.getString("id"))
    }
}
```

**新（Navigation 2.8+）：**
```kotlin
@Serializable data class Home
@Serializable data class Details(val id: String)

NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen() }
    composable<Details> { backStackEntry ->
        val args: Details = backStackEntry.toRoute()
        DetailsScreen(id = args.id)
    }
}
```

**迁移说明：** 类型安全路由消除字符串拼写错误和运行时崩溃。需要 `kotlinx-serialization` 插件和 `navigation-compose:2.8.0+`。使用自定义序列化器编码复杂对象。

---

## `accompanist-systemuicontroller` → `enableEdgeToEdge()`

**旧：**
```kotlin
val systemUiController = rememberSystemUiController()
systemUiController.setSystemBarsColor(
    color = Color.Transparent,
    darkIcons = false
)
```

**新（Compose 1.7+）：**
```kotlin
enableEdgeToEdge()
// 在 Activity.onCreate() 中 setContent {} 之前调用
```

**迁移说明：** Compose 1.7 起内置。自动处理状态栏、导航栏和 IME 背后内容。完全移除 `accompanist-systemuicontroller` 依赖。

---

## `accompanist-pager` → `HorizontalPager`/`VerticalPager`

**旧：**
```kotlin
val pagerState = rememberPagerState()
HorizontalPager(count = items.size, state = pagerState) { page ->
    PageContent(items[page])
}
```

**新（Foundation）：**
```kotlin
val pagerState = rememberPagerState(pageCount = { items.size })
HorizontalPager(state = pagerState) { page ->
    PageContent(items[page])
}
```

**迁移说明：** `foundation:1.6+` 中的原生 Pager 替代 accompanist。移除外部依赖。状态初始化略有不同；通过 lambda 传递动态页面数。

---

## `accompanist-swiperefresh` → `PullToRefreshBox`

**旧：**
```kotlin
SwipeRefresh(state = rememberSwipeRefreshState(isRefreshing), onRefresh = { load() }) {
    LazyColumn { items(data) { item -> ItemRow(item) } }
}
```

**新（Material3）：**
```kotlin
PullToRefreshBox(isRefreshing = isRefreshing, onRefresh = { load() }) {
    LazyColumn { items(data) { item -> ItemRow(item) } }
}
```

**迁移说明：** `material3:1.2+` 中的 `PullToRefreshBox` 是官方替代。更简洁的 API。移除 `accompanist-swiperefresh` 依赖。

---

## `accompanist-flowlayout` → `FlowRow`/`FlowColumn`

**旧：**
```kotlin
FlowRow(mainAxisSize = SizeMode.Expand) {
    items.forEach { item -> Chip(text = item) }
}
```

**新（Foundation）：**
```kotlin
FlowRow(modifier = Modifier.fillMaxWidth()) {
    items.forEach { item -> Chip(text = item) }
}
```

**迁移说明：** `foundation:1.6+` 中的 FlowRow/FlowColumn。API 简化；使用标准 modifier 替代 `SizeMode`。性能更好，内存开销更小。

---

## `LazyColumn { animateItemPlacement() }` → `LazyColumn { animateItem() }`

**旧：**
```kotlin
LazyColumn {
    items(items, key = { it.id }) { item ->
        ItemRow(item.name, Modifier.animateItemPlacement())
    }
}
```

**新：**
```kotlin
LazyColumn {
    items(items, key = { it.id }) { item ->
        ItemRow(item.name, Modifier.animateItem())
    }
}
```

**迁移说明：** `animateItem()` 是现代 API（Compose 1.7+）。返回动画状态以进行更精细的控制。`animateItemPlacement()` 仍能工作但已被取代。

---

## `Modifier.composed` 模式 → `Modifier.Node` API

**旧：**
```kotlin
fun Modifier.myModifier(value: Int) = composed {
    val state = remember { mutableStateOf(value) }
    Modifier.fillMaxWidth().padding(8.dp)
}
```

**新：**
```kotlin
fun Modifier.myModifier(value: Int) = this.then(
    Modifier
        .fillMaxWidth()
        .padding(8.dp)
)
// 或对于复杂状态：
class MyModifierNode(val value: Int) : ModifierNodeElement<MyNodeImpl>() {
    override fun create() = MyNodeImpl(value)
    override fun update(node: MyNodeImpl) { node.value = value }
}
private class MyNodeImpl(var value: Int) : Modifier.Node
```

**迁移说明：** `composed {}` 有开销；如果不需要 `remember` 调用则避免使用。对于有状态的 modifier，优先使用 `ModifierNode` API（Compose 1.8+）。迁移现有代码前先做基准测试。

---

## 原始类型状态优化：`mutableStateOf(0)` → `mutableIntStateOf(0)`

**旧：**
```kotlin
var count by remember { mutableStateOf(0) }
var temperature by remember { mutableStateOf(37.5f) }
```

**新：**
```kotlin
var count by remember { mutableIntStateOf(0) }
var temperature by remember { mutableFloatStateOf(37.5f) }
```

**迁移说明：** 原始类型专用函数（`mutableIntStateOf`、`mutableFloatStateOf`、`mutableLongStateOf`）避免装箱。在 UI 代码中性能影响可忽略，但自 Compose 1.4+ 起是最佳实践。

---

## `collectAsState()` → `collectAsStateWithLifecycle()`

**旧：**
```kotlin
val state by viewModel.uiState.collectAsState()
```

**新：**
```kotlin
val state by viewModel.uiState.collectAsStateWithLifecycle()
```

**迁移说明：** `collectAsStateWithLifecycle()`（Compose 1.6+）尊重生命周期 — activity 暂停时自动停止收集。防止内存泄漏和冗余工作。需要 `androidx.lifecycle:lifecycle-runtime-compose`。

---

## `@ExperimentalMaterial3Api` 毕业

**旧：**
```kotlin
@OptIn(ExperimentalMaterial3Api::class)
fun MyScreen() {
    DatePicker(state = rememberDatePickerState())
}
```

**新（Compose 1.8+, Material3 1.3+）：**
```kotlin
fun MyScreen() {
    DatePicker(state = rememberDatePickerState())
}
```

**迁移说明：** DatePicker、TimePicker、ExposedDropdownMenuBox 和 SearchBar 在 Material3 1.3+ 中已稳定。移除 `@OptIn` 注解。API 已稳定 — 可安全用于生产。

---

## `Scaffold` 内边距强制使用

**旧（有问题）：**
```kotlin
Scaffold(topBar = { TopAppBar() }) {
    LazyColumn { items(data) { item -> ItemRow(item) } }
}
```

**新（Compose 1.6+ 起强制）：**
```kotlin
Scaffold(topBar = { TopAppBar() }) { innerPadding ->
    LazyColumn(modifier = Modifier.padding(innerPadding)) {
        items(data) { item -> ItemRow(item) }
    }
}
```

**迁移说明：** 自 Compose 1.6 起必须使用 `innerPadding` 参数。忽略它会导致内容被系统栏重叠。编译器现在强制这一点 — 旧模式无法编译。

---

## Material 2 → Material 3 迁移

**旧（Material）：**
```kotlin
Button(onClick = { }) { Text("Click") }
TextField(value = text, onValueChange = { text = it })
Surface(color = MaterialTheme.colors.primary) { /* */ }
```

**新（Material3）：**
```kotlin
Button(onClick = { }) { Text("Click") }  // 相同签名
TextField(value = text, onValueChange = { text = it })  // 相同签名
Surface(color = MaterialTheme.colorScheme.primary) { /* */ }
```

**迁移说明：** 大多数 Composable API 兼容。主要变化：`colors` → `colorScheme`、新形状系统、更新的涟漪默认行为。使用 Compose BOM 对齐 Material3 版本。

---

## `WindowInsets` 与 Edge-to-Edge

**旧：**
```kotlin
Surface(modifier = Modifier.systemBarsPadding()) { /* */ }
```

**新（API 35+ 默认 edge-to-edge）：**
```kotlin
Surface(modifier = Modifier.padding(WindowInsets.systemBars.asPaddingValues())) { /* */ }
// 或在 Activity 中使用 enableEdgeToEdge() — 自动处理
```

**迁移说明：** Edge-to-edge 在 Android 15+ 上是默认的。系统栏颜色由 `enableEdgeToEdge()` 管理。使用 `WindowInsets.safeDrawing` 进行刘海屏感知布局。弃用手动 `systemBarsPadding()` 调用。

---

## `ObservableState` 模式变更

**旧：**
```kotlin
@Composable
fun observe(state: ObservableState): State<T> = produceState(state.value) {
    state.onChange { value = it }
}
```

**新：**
```kotlin
@Composable
fun <T> ObservableState<T>.asState(): State<T> = produceState(this.value) {
    snapshotFlow { value }.collect { value = it }
}
```

**迁移说明：** `snapshotFlow {}` 优先于直接监听器（Compose 1.6+）。与 Compose 的快照系统集成更好。使用 `distinctUntilChanged()` 避免冗余重组。
