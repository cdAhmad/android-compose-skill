# 性能优化参考

## 三个阶段：组合、布局、绘制

每帧由三个阶段组成。理解每个阶段中的状态读取可防止不必要的重组。

### 组合阶段
- 执行 composable 函数，评估状态读取
- 生成 lambda 和实例分配
- **此处的状态读取会触发整个作用域的重组**

### 布局阶段
- 计算尺寸和位置，运行 `measure` 和 `layout` 块
- 可以读取状态而不触发组合重组
- 可变状态读取可行；优先使用 `Modifier.offset { }` 而非 `Modifier.offset()`

### 绘制阶段
- 发出绘制操作，运行 `Canvas` 和自定义 `DrawScope`
- 读取可变状态会产生稳定性警告

**来源**：`androidx/compose/runtime/Composer.kt`

---

## 使用编译器报告跳过重组

Compose 编译器生成 `$changed` 位掩码来检测状态变化。启用编译器报告以检查稳定性和可跳过性：

```kotlin
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

构建后（`./gradlew assembleRelease`），检查 `build/compose_reports/` 中生成的文件：

- **`*_composables.txt`** — 显示每个 composable 的可重启性和可跳过性：
  ```
  restartable skippable fun MyComponent(name: String, onClick: Function0<Unit>)
  restartable fun UnstableComponent(items: List<Item>)  // 不可跳过 — 不稳定参数
  ```

- **`*_classes.txt`** — 显示每个类的稳定性推断：
  ```
  stable class User { stable val name: String }
  unstable class ScreenState { unstable val items: List<Item> }
  ```

缺少 `skippable` 的 composable 意味着编译器无法在重组期间跳过它，即使输入未变。通过稳定其参数来修复（见下方稳定性部分）。

### 稳定性 — @Stable 和 @Immutable

如果类型满足以下条件，则它是**稳定的**：
- 其 public 属性是稳定的
- 重写的 `equals()` 和 `hashCode()` 基于稳定属性
- 传递相同实例时跳过重组

显式标记稳定类型：

```kotlin
@Immutable
data class Person(val name: String, val age: Int)

@Stable
class UserViewModel : ViewModel {
    private val _state = MutableState(UserState())
    val state: State<UserState> = _state
}

// 接收稳定类型的 composable 可以跳过重组
@Composable
fun PersonCard(person: Person) {
    Text(person.name)  // 如果 person 未变则跳过
}
```

**避免**：在具有可变字段或非 final 属性的数据类上使用 `@Stable`。

---

## 强跳过模式（默认）

Android Gradle Plugin 8.0+ 和 Compose 编译器 1.5.0+ 启用**强跳过模式**。这改变了 lambda 的处理方式：

未启用强跳过时，每个 lambda 都是不稳定的。启用后：
- 如果所有捕获变量都是稳定的，lambda 变为稳定
- 减少不必要的重组

```kotlin
// 启用强跳过时：如果 count 是稳定的，lambda 是稳定的
@Composable
fun Counter(count: Int) {
    Button(onClick = { println(count) }) {  // 稳定 lambda
        Text("Count: $count")
    }
}
```

检查 `build.gradle.kts`：
```kotlin
android {
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.0"
    }
}
```

---

## 将状态读取推迟到布局/绘制阶段

在组合中读取状态会触发重组。将读取推后到更晚的阶段：

### 糟糕：每次偏移变化都重组
```kotlin
@Composable
fun Box(offsetX: State<Float>) {
    val x = offsetX.value  // 在组合中读取，触发重组
    Box(modifier = Modifier.offset(x.dp, 0.dp))
}
```

### 良好：在布局阶段推迟读取
```kotlin
@Composable
fun Box(offsetX: State<Float>) {
    Box(
        modifier = Modifier.offset {
            IntOffset(offsetX.value.toInt(), 0)  // 在布局阶段读取
        }
    )
}
```

对状态依赖的定位使用 `Modifier.offset { }`（lambda）而非 `Modifier.offset()`（参数）。

---

## derivedStateOf — 减少重组频率

从状态派生昂贵计算时，用 `derivedStateOf` 包装以去重重组：

```kotlin
// 糟糕：每次 items 变化都重组
@Composable
fun SearchResults(items: List<Item>, query: String) {
    val filtered = items.filter { query in it.title }  // 组合阶段
    LazyColumn {
        items(filtered) { /* ... */ }
    }
}

// 良好：仅在实际过滤结果变化时才重组
@Composable
fun SearchResults(items: List<Item>, query: String) {
    val filtered = remember(items, query) {
        derivedStateOf { items.filter { query in it.title } }
    }
    LazyColumn {
        items(filtered.value) { /* ... */ }
    }
}
```

`derivedStateOf` 去重下游重组 — 两次不同的过滤产生相同列表时，只触发一次下游重组。

---

## 带 Key 的 remember

避免不必要的重新计算：

```kotlin
// 每次重组都重新计算
@Composable
fun ExpensiveItem(id: Int) {
    val metadata = computeMetadata(id)  // 每次调用
    Text(metadata)
}

// 仅在 id 变化时重新计算
@Composable
fun ExpensiveItem(id: Int) {
    val metadata = remember(id) { computeMetadata(id) }
    Text(metadata)
}

// 多个 key
@Composable
fun Item(id: Int, userId: Int) {
    val data = remember(id, userId) { fetchData(id, userId) }
    Text(data.toString())
}
```

如果计算廉价（字符串格式化、简单对象），省略 `remember`。过度包裹会导致内存泄漏。

---

## LazyList 性能 — Key 和 ContentType

### 始终提供 Key

Key 启用条目复用和动画：

```kotlin
// 糟糕：无 key，列表变化时条目重新创建
LazyColumn {
    items(users) { user ->
        UserRow(user)
    }
}

// 良好：key 启用复用
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserRow(user)
    }
}
```

### ContentType 用于高效复用

```kotlin
sealed class ListItem {
    data class Header(val title: String) : ListItem()
    data class User(val user: User) : ListItem()
}

LazyColumn {
    items(
        items = items,
        key = { it.hashCode() },
        contentType = { item ->
            when (item) {
                is ListItem.Header -> "header"
                is ListItem.User -> "user"
            }
        }
    ) { item ->
        when (item) {
            is ListItem.Header -> HeaderRow(item.title)
            is ListItem.User -> UserRow(item.user)
        }
    }
}
```

没有 `contentType`，所有条目竞争一个 ViewHolder 池。有了它，条目高效复用。

### 避免在条目作用域中分配

```kotlin
// 糟糕：每次重组都分配
LazyColumn {
    items(users) { user ->
        val userState = remember { mutableStateOf(user) }
        UserRow(userState.value)
    }
}

// 良好：只分配一次
LazyColumn {
    items(
        items = users,
        key = { it.id }
    ) { user ->
        UserRow(user)
    }
}
```

---

## Baseline Profiles

Baseline profiles 指导 R8 预编译热代码路径，减少启动时间和卡顿。

### 生成 Profiles

使用 Jetpack Macrobenchmark 记录 profiles：

```kotlin
@RunWith(AndroidBenchmarkRunner::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startupCompilation() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimings.FIRST_FRAME),
        iterations = 10,
        setupBlock = {
            pressHome()
            startActivityAndWait()
        }
    ) {
        // 与应用交互
    }
}
```

Profiles 生成在 `baseline-prof.txt` 中：
```
androidx/compose/runtime/Recomposer;startRecomposition()V
com/example/MyScreen;ComposableFunctionName(ILandroidx/compose/runtime/Composer;I)V
```

---

## R8/ProGuard Compose 规则

Compose 包含默认 ProGuard 规则。确保 `shrinkResources true` 和 `minifyEnabled true`：

```gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

保留稳定性的自定义规则：

```proguard
-keep @androidx.compose.runtime.Stable class **
-keep @androidx.compose.runtime.Immutable class **
-keepclassmembers class * {
    @androidx.compose.runtime.Stable <methods>;
}
```

---

## 测量性能

### Layout Inspector — 重组计数

在 Android Studio 中：
1. 在设备上运行应用
2. 打开 **Layout Inspector**（Tools > Layout Inspector）
3. 选择目标进程
4. 勾选 **Show Composition Counts**（inspector 中的切换）

重组计数显示自检查开始以来每个 composable 被重组的次数。

### Macrobenchmark — 帧时间

```kotlin
benchmarkRule.measureRepeated(
    packageName = "com.example.app",
    metrics = listOf(FrameTimingMetric()),
    iterations = 10
) {
    // 交互：滚动、点击等
}
```

报告帧时间（ms）、卡顿、抖动。60 fps 目标 <16.67ms。

---

## 常见热路径

### 组合中的字符串格式化
```kotlin
// 糟糕：每次重组都分配字符串
@Composable
fun Counter(count: Int) {
    Text("Count: ${count}")  // 调用 String.format
}

// 仍组合，但已优化
@Composable
fun Counter(count: Int) {
    Text(buildString { append("Count: "); append(count) })
}
```

### 无 derivedStateOf 的列表过滤
```kotlin
// 糟糕：每次重组都过滤
@Composable
fun FilteredList(items: List<Item>, predicate: (Item) -> Boolean) {
    LazyColumn {
        items(items.filter(predicate)) { /* ... */ }
    }
}

// 良好
@Composable
fun FilteredList(items: List<Item>, predicate: (Item) -> Boolean) {
    val filtered = remember(items, predicate) {
        derivedStateOf { items.filter(predicate) }
    }
    LazyColumn {
        items(filtered.value) { /* ... */ }
    }
}
```

### 在 Lambda 中创建对象
```kotlin
// 糟糕
Button(
    colors = ButtonDefaults.buttonColors(
        containerColor = if (isPressed) Color.Red else Color.Blue
    )
) { }

// 良好：只计算一次
val buttonColors = remember(isPressed) {
    ButtonDefaults.buttonColors(
        containerColor = if (isPressed) Color.Red else Color.Blue
    )
}
Button(colors = buttonColors) { }
```

---

## 反模式

### 把所有东西包裹在 remember 中
```kotlin
// 不必要
val text = remember { "Hello" }
val size = remember { 12.sp }
val color = remember { Color.Black }
```

`remember` 仅用于可变状态或昂贵计算。

### 过早优化
先分析。没有 Layout Inspector 数据就不要添加 `derivedStateOf` 或 `remember`。

### 在可变数据类上使用 @Stable
```kotlin
// 不要
@Stable
data class MutableUser(val name: String, val age: MutableState<Int>)

// 要
@Immutable
data class User(val name: String, val age: Int)
```

---

## 资源

- **Compose Compiler Reports**: https://developer.android.com/develop/ui/compose/performance/stability-report
- **Macrobenchmark**: https://developer.android.com/develop/ui/compose/performance/measurement
- **Baseline Profiles**: https://developer.android.com/develop/ui/compose/performance/baseline-profiles

---

## 零尺寸 DrawScope 防护

在初始组合期间，composable 的尺寸可能为零。这会导致 `size.minDimension / 2` 等计算崩溃。

```kotlin
// BAD: 尺寸为零时崩溃
Canvas(modifier = Modifier.fillMaxSize()) {
    val radius = size.minDimension / 2  // NaN 或除零
    drawCircle(color = Color.Blue, radius = radius)
}

// GOOD: 始终防护
Canvas(modifier = Modifier.fillMaxSize()) {
    if (size.minDimension <= 0f) return@Canvas
    val radius = size.minDimension / 2
    drawCircle(color = Color.Blue, radius = radius)
}
```

规则：永远不要在没有显式高度约束的情况下对 Canvas 使用 `fillMaxSize()`。始终防护 DrawScope 操作。

---

## Composition Tracing

使用 `trace()` 进行 Perfetto/systrace 集成：

```kotlin
@Composable
fun ExpensiveScreen() {
    trace("ExpensiveScreen") {
        // composable 主体 — 在系统追踪中可见
    }
}
```

无需添加日志即可在生产环境分析中识别慢 composable。

---

## movableContentOf

在布局位置之间移动内容时避免重组：

```kotlin
val movableContent = remember {
    movableContentOf {
        ExpensiveChild() // 只组合一次，移动而不重组
    }
}

if (isExpanded) {
    ExpandedLayout { movableContent() }
} else {
    CollapsedLayout { movableContent() }
}
```

没有 `movableContentOf`，切换布局会销毁并重新组合 `ExpensiveChild`。有了它，内容被移动，保留状态并避免重组。

---

## 编译器报告（扩展）

启用编译器报告以查看哪些 composable 可跳过、哪些类型稳定：

```kotlin
// build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

构建后，检查输出文件：

- `*_composables.txt` — 显示每个 composable 的状态：
  ```
  restartable skippable fun MyComponent(name: String, onClick: Function0<Unit>)
  restartable fun UnstableComponent(items: List<Item>)  // 不可跳过！
  ```

- `*_classes.txt` — 显示类型稳定性：
  ```
  stable class User { stable val name: String }
  unstable class ScreenState { unstable val items: List<Item> }
  ```

通过以下方式修复不稳定类型：
1. 在类上使用 `@Stable` 注解
2. 使用 `kotlinx.collections.immutable.ImmutableList` 替代 `List`
3. 在多模块项目中将其添加到 `compose-stability-config.txt`

---

## 生产环境性能规则

1. **R8：在 release 中剥离 preview + semantics** — 添加到 `proguard-rules.pro`：
```
-assumenosideeffects class androidx.compose.ui.tooling.preview.** { *; }
```

2. **`@Suppress("ComposeUnstableCollections")`** — 当稳定性不值得复杂性时务实跳过：
```kotlin
@Suppress("ComposeUnstableCollections")
@Composable
fun ItemList(items: List<Item>) { // List 不稳定但此处可接受
    // ...
}
```

3. **ImmutableList 用于稳定性** — Guava `ImmutableList` 或 `kotlinx.collections.immutable`：
```kotlin
// 使参数稳定，启用重组跳过
@Composable
fun StableList(items: ImmutableList<Item>) { ... }
```

4. **ReportDrawnWhen 用于启动性能**：
```kotlin
ReportDrawnWhen { items.isNotEmpty() }
```

5. **Canvas 始终显式定尺寸** — 永远不要无高度约束的 `fillMaxSize()`：
```kotlin
// BAD
Canvas(Modifier.fillMaxSize()) { ... }

// GOOD
Canvas(Modifier.fillMaxWidth().height(200.dp)) { ... }
```
