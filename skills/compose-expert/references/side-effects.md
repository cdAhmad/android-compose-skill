# Jetpack Compose 副作用参考

Compose 是声明式的，但应用必须与命令式世界交互：启动协程、注册监听器、管理资源。副作用是桥梁。理解何时以及如何使用它们对正确性至关重要。

## 副作用心智模型

Compose 在状态变化时重组。副作用是在正常组合和重组周期之外运行的代码块：

- **组合**：计算 UI 树
- **副作用**：运行命令式代码（协程、回调、生命周期事件）
- **布局**：测量和定位元素
- **绘制**：渲染到屏幕

副作用在组合成功*之后*运行。如果组合失败，副作用不会运行。

```kotlin
@Composable
fun MyScreen() {
    // 这在组合期间运行
    val state = remember { mutableStateOf("initial") }

    // 这在组合之后运行，且仅在 'state.value' 变化时运行
    LaunchedEffect(state.value) {
        println("State changed to: ${state.value}")
    }

    // 这在每次组合之后运行（谨慎使用）
    SideEffect {
        println("Recomposition happened")
    }

    // 这在 composable 离开组合时运行
    DisposableEffect(Unit) {
        onDispose {
            println("Composable is leaving composition")
        }
    }

    Button(onClick = { state.value = "updated" }) {
        Text(state.value)
    }
}
```

## SideEffect — 每次成功组合之后

`SideEffect` 在*每次*成功组合之后运行。它无清理、无 key，始终执行。

```kotlin
@Composable
fun MyComposable() {
    var clickCount by remember { mutableStateOf(0) }

    // 每次重组后运行
    SideEffect {
        println("Recomposed! Click count: $clickCount")
    }

    Button(onClick = { clickCount++ }) {
        Text("Clicks: $clickCount")
    }
}
```

### 使用场景

- 将 Compose 状态与外部系统同步（例如 Analytics 日志记录）
- 更新非 Compose UI 元素
- 不需要清理的单向同步

```kotlin
@Composable
fun TrackScreenView(screenName: String) {
    SideEffect {
        Analytics.logScreenView(screenName)
    }
}
```

**应做：** 用于简单的、无状态的同步。
**不应做：** 用于资源分配（改用 `DisposableEffect`）。

来源：`compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Effects.kt`

## LaunchedEffect(key) — 组合作用域内的协程

`LaunchedEffect` 在与 composable 生命周期绑定的作用域中启动协程。如果 key 变化或 composable 离开组合，协程会被取消。

```kotlin
@Composable
fun DataLoader(userId: String) {
    var data by remember { mutableStateOf<String?>(null) }

    // 当 userId 变化或 composable 进入组合时运行协程
    LaunchedEffect(userId) {
        data = loadData(userId)  // 挂起函数
    }

    Text(data ?: "Loading...")
}
```

### Key 选择

```kotlin
// Key = Unit：在 composable 进入组合时运行一次，永不取消/重启
LaunchedEffect(Unit) {
    setupOnce()
}

// Key = 特定值：值变化时重新运行
var userId by remember { mutableStateOf("user1") }
LaunchedEffect(userId) {
    loadUserData(userId)  // userId 变化时重新运行
}

// 多个 key：任一 key 变化时重新运行
LaunchedEffect(userId, postId) {
    loadUserAndPost(userId, postId)
}

// 无 key 参数（不推荐）：等价于 Unit
LaunchedEffect {
    setupOnce()
}
```

### 常见错误：错误的 Key 选择

```kotlin
// 不要：Key 每次重组都变化（创建无限循环）
@Composable
fun BadKeySelection() {
    var count by remember { mutableStateOf(0) }
    val randomKey = Random.nextInt()  // 每次重组都变化！

    LaunchedEffect(randomKey) {
        count++  // 这会无限启动
    }

    Text("Count: $count")
}

// 要：使用代表你所依赖数据的稳定 key
@Composable
fun GoodKeySelection(userId: String) {
    var userData by remember { mutableStateOf<User?>(null) }

    LaunchedEffect(userId) {
        userData = fetchUser(userId)
    }

    Text(userData?.name ?: "Loading...")
}
```

### 取消行为

```kotlin
@Composable
fun ResourceUser(shouldLoad: Boolean) {
    LaunchedEffect(shouldLoad) {
        if (shouldLoad) {
            val resource = acquireResource()
            try {
                delay(5000)  // 长时间操作
                processResource(resource)
            } finally {
                resource.close()  // 即使取消也会运行
            }
        }
    }
}

// 如果 shouldLoad 变为 false，LaunchedEffect 协程被取消。
// finally 块确保清理。
```

## DisposableEffect(key) — 用于清理

`DisposableEffect` 在组合之后运行，并要求提供清理函数（onDispose）。用于监听器、注册和资源。

```kotlin
@Composable
fun LocationListener(context: Context) {
    DisposableEffect(context) {
        val listener = LocationListener { location ->
            println("Location: $location")
        }
        // 注册监听器
        locationManager.requestLocationUpdates(
            LocationManager.GPS_PROVIDER,
            0,
            0f,
            listener
        )

        // 清理：注销监听器
        onDispose {
            locationManager.removeUpdates(listener)
        }
    }
}
```

### 常见模式：生命周期事件

```kotlin
@Composable
fun ScreenWithLifecycle() {
    val lifecycle = LocalLifecycleOwner.current.lifecycle

    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> println("Screen resumed")
                Lifecycle.Event.ON_PAUSE -> println("Screen paused")
                else -> {}
            }
        }
        lifecycle.addObserver(observer)

        onDispose {
            lifecycle.removeObserver(observer)
        }
    }
}
```

**应做：** 为每个分配的资源使用 `DisposableEffect`。
**不应做：** 忘记 `onDispose` 块（会导致资源泄漏）。

来源：`compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Effects.kt`

## rememberCoroutineScope — 从事件处理器启动

`rememberCoroutineScope` 提供一个与 composable 绑定的协程作用域。用于从事件处理器（点击、手势）中启动协程。

```kotlin
@Composable
fun ButtonWithAsync() {
    val scope = rememberCoroutineScope()
    var result by remember { mutableStateOf("") }

    Button(
        onClick = {
            // 从点击处理器中启动协程
            scope.launch {
                result = fetchData()
            }
        }
    ) {
        Text("Fetch")
    }

    Text(result)
}
```

### 应做 vs 不应做

```kotlin
// 不要：普通函数作用域无效
@Composable
fun BadAsync() {
    var result by remember { mutableStateOf("") }

    Button(
        onClick = {
            runBlocking {  // 阻塞 UI 线程！
                result = fetchData()
            }
        }
    ) {
        Text("Fetch")
    }
}

// 要：使用 rememberCoroutineScope
@Composable
fun GoodAsync() {
    val scope = rememberCoroutineScope()
    var result by remember { mutableStateOf("") }

    Button(
        onClick = {
            scope.launch {
                result = fetchData()
            }
        }
    ) {
        Text("Fetch")
    }
}
```

## rememberUpdatedState — 捕获最新值

长时间运行的副作用需要频繁变化状态的最新值，但你不想在每次变化时重启副作用。

```kotlin
// 不要：callback 变化时效果重启
@Composable
fun BadCallback(onSuccess: (String) -> Unit) {
    LaunchedEffect(onSuccess) {  // callback 变化时重启！
        val result = expensiveOperation()
        onSuccess(result)
    }
}

// 要：使用 rememberUpdatedState 在不重启的情况下捕获最新值
@Composable
fun GoodCallback(onSuccess: (String) -> Unit) {
    val updatedOnSuccess = rememberUpdatedState(onSuccess)

    LaunchedEffect(Unit) {
        val result = expensiveOperation()
        updatedOnSuccess.value(result)
    }
}
```

### 另一个示例：动画

```kotlin
@Composable
fun AnimateWithCallback(
    shouldAnimate: Boolean,
    onAnimationEnd: () -> Unit
) {
    val updatedCallback = rememberUpdatedState(onAnimationEnd)
    var progress by remember { mutableStateOf(0f) }

    LaunchedEffect(shouldAnimate) {
        if (shouldAnimate) {
            while (progress < 1f) {
                progress += 0.1f
                delay(16)
            }
            updatedCallback.value()  // 不重启就调用最新 callback
        }
    }
}
```

## produceState — 将非 Compose 状态转换为 Compose 状态

`produceState` 将命令式状态源（回调、flow、协程）转换为 Compose 状态。

```kotlin
@Composable
fun UserData(userId: String): State<User?> = produceState<User?>(initialValue = null) {
    value = fetchUser(userId)

    // 可选：用于生命周期清理
    snapshotFlow { userId }.collect { newUserId ->
        value = fetchUser(newUserId)
    }
}

// 使用
@Composable
fun UserScreen(userId: String) {
    val user by UserData(userId)
    Text(user?.name ?: "Loading...")
}
```

### 与 Flow 集成

```kotlin
@Composable
fun <T> Flow<T>.collectAsState(initial: T): State<T> = produceState(initial) {
    collect { value = it }
}

// 使用
@Composable
fun ObserveFlow(dataFlow: Flow<String>) {
    val data by dataFlow.collectAsState(initial = "")
    Text(data)
}
```

## 副作用顺序和生命周期

副作用在组合之后按声明顺序执行：

```kotlin
@Composable
fun EffectOrder() {
    println("1. Composition")

    SideEffect {
        println("4. Side effect (after every composition)")
    }

    LaunchedEffect(Unit) {
        println("3. Launched effect (async, but scheduled)")
        delay(100)
        println("5. After delay in launched effect")
    }

    DisposableEffect(Unit) {
        println("2. Disposable effect setup (after composition)")

        onDispose {
            println("6. Cleanup when leaving composition")
        }
    }

    println("End of composition body")
}

// 输出顺序（大致）：
// 1. Composition
// End of composition body
// 2. Disposable effect setup (after composition)
// 3. Launched effect (async, but scheduled)
// 4. Side effect (after every composition)
// 5. After delay in launched effect
// [... later when composable leaves ...]
// 6. Cleanup when leaving composition
```

## 常见错误

### 当 Key 应该变化时使用 LaunchedEffect(Unit)

```kotlin
// 不要：效果只运行一次，永不更新
@Composable
fun BadSearch(query: String) {
    var results by remember { mutableStateOf<List<String>>(emptyList()) }

    LaunchedEffect(Unit) {
        results = search(query)  // 只运行一次！
    }

    Text("Results: ${results.size}")
}

// 要：使用 query 作为 key
@Composable
fun GoodSearch(query: String) {
    var results by remember { mutableStateOf<List<String>>(emptyList()) }

    LaunchedEffect(query) {
        results = search(query)  // query 变化时重新运行
    }

    Text("Results: ${results.size}")
}
```

### 在 DisposableEffect 中忘记清理

```kotlin
// 不要：内存泄漏
@Composable
fun BadListener(context: Context) {
    DisposableEffect(Unit) {
        val listener = MyListener()
        context.registerListener(listener)
        // 缺少：onDispose { context.unregisterListener(listener) }
    }
}

// 要：始终清理
@Composable
fun GoodListener(context: Context) {
    DisposableEffect(Unit) {
        val listener = MyListener()
        context.registerListener(listener)

        onDispose {
            context.unregisterListener(listener)
        }
    }
}
```

### 直接捕获可变状态

```kotlin
// 不要：效果中的状态过期
@Composable
fun BadCapture() {
    var count by remember { mutableStateOf(0) }

    LaunchedEffect(Unit) {
        delay(1000)
        println(count)  // 可能已过期！
    }

    Button(onClick = { count++ }) { Text("Click") }
}

// 要：使用 rememberUpdatedState 或将其包含在 key 中
@Composable
fun GoodCapture() {
    var count by remember { mutableStateOf(0) }

    val updatedCount = rememberUpdatedState(count)
    LaunchedEffect(Unit) {
        delay(1000)
        println(updatedCount.value)  // 始终是最新值
    }

    Button(onClick = { count++ }) { Text("Click") }
}
```

---

**总结：** 副作用桥接声明式 Compose 与命令式系统。掌握 `LaunchedEffect` 中的 key 选择，始终在 `DisposableEffect` 中清理，对长时间运行的副作用使用 `rememberUpdatedState` 获取最新值，并优先选择基于 effect 的模式而非手动生命周期管理。
