# CompositionLocals：Jetpack Compose 中的隐式数据传递

CompositionLocals 提供了一种在组合树中隐式传递数据的方式，无需通过每个函数参数逐层传递。

## 什么是 CompositionLocals？

CompositionLocal 是组合中的一个槽位，用于存储任何后代 composable 都可以访问的值，无需显式参数传递。值通过 `CompositionLocalProvider` 提供，通过 `current` 访问。

```kotlin
val localAppTheme = compositionLocalOf { "Light" }

@Composable
fun MyScreen() {
  CompositionLocalProvider(localAppTheme provides "Dark") {
    DescendantComposable() // 可通过 localAppTheme.current 访问 "Dark"
  }
}

@Composable
fun DescendantComposable() {
  Text(localAppTheme.current) // 读取 "Dark"
}
```

**来源：** `androidx/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/CompositionLocal.kt`

## compositionLocalOf vs staticCompositionLocalOf

关键区别在于值变化时**何时触发重组**。

### compositionLocalOf
值变化时会导致所有后代重组。适用于子组件真正依赖该值的场景。

```kotlin
val LocalUserPreferences = compositionLocalOf { UserPreferences() }
```

**重组行为：** 所有消费者都会重组。

### staticCompositionLocalOf
不会触发后代重组；仅直接影响读取者。适用于**确信后代不依赖更新**或更新很少发生的场景。

```kotlin
val LocalAppVersion = staticCompositionLocalOf { "1.0.0" }
```

**⚠️ 陷阱：** 如果子组件读取了 `LocalAppVersion.current` 并期望获得更新，将得到过期数据。仅用于真正静态的配置。

### compositionLocalWithComputedDefaultOf
用于计算默认值。当没有活跃的 provider 时，每次读取都会调用该 lambda。

```kotlin
val LocalResources = compositionLocalWithComputedDefaultOf { context.resources }
```

这比 `compositionLocalOf { lazy { ... } }` 更高效，因为它避免了不必要的状态捕获。

## 内置 CompositionLocals

Compose runtime 和 UI 库提供了标准 locals：

| Local | 类型 | 用途 |
|-------|------|------|
| `LocalContext` | `Context` | Android Context（需要 AndroidCompositionLocals） |
| `LocalConfiguration` | `Configuration` | 屏幕尺寸、方向、密度 |
| `LocalDensity` | `Density` | 像素密度，用于 dp/px 转换 |
| `LocalLayoutDirection` | `LayoutDirection` | LTR/RTL 方向 |
| `LocalView` | `View` | 底层 Android View（如可用） |
| `LocalLifecycleOwner` | `LifecycleOwner` | Activity/Fragment 生命周期 |
| `LocalSavedStateRegistryOwner` | `SavedStateRegistryOwner` | 用于状态持久化 |

**来源：** `androidx/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidCompositionLocals.android.kt`

```kotlin
@Composable
fun MyComposable() {
  val context = LocalContext.current
  val density = LocalDensity.current
  val config = LocalConfiguration.current

  Text("屏幕宽度: ${config.screenWidthDp}dp")
}
```

## 通过 CompositionLocalProvider 提供值

提供一个或多个 local 值：

```kotlin
// 单个 local
CompositionLocalProvider(LocalUserPreferences provides user) {
  Content()
}

// 多个 locals
CompositionLocalProvider(
  LocalUserPreferences provides user,
  LocalTheme provides darkTheme,
  LocalLanguage provides "zh"
) {
  Content()
}
```

值仅对**后代**生效：

```kotlin
CompositionLocalProvider(LocalUserPreferences provides userA) {
  ComponentA() // 读取到 userA
  CompositionLocalProvider(LocalUserPreferences provides userB) {
    ComponentB() // 读取到 userB（覆盖）
  }
  ComponentC() // 读取到 userA（原始值）
}
```

## 创建自定义 CompositionLocals

在顶层、composable 函数外部创建 locals：

```kotlin
data class AppTheme(val isDark: Boolean, val colors: Colors)

val LocalAppTheme = compositionLocalOf<AppTheme> {
  error("AppTheme 未提供")
}

// 可空默认值
val LocalOptionalUser = compositionLocalOf<User?> { null }
```

**何时创建 CompositionLocal：**
- 值被许多后代需要
- 作为参数传递会导致 "prop drilling"
- 值属于配置类（主题、区域设置、权限）

**何时不使用 CompositionLocal：**
- 仅 1–2 层 composable 需要它 → 使用参数
- 值频繁变化且子组件需要精确控制 → 使用 State/ViewModel
- 它是应该可测试的依赖项 → 优先使用参数或依赖注入

## 使用 CompositionLocals 进行测试

提供测试替身以避免真实实现：

```kotlin
@Composable
fun MyScreen() {
  val user = LocalUserRepository.current
  Text(user.name)
}

// 测试中
@Test
fun testMyScreen() {
  composeRule.setContent {
    CompositionLocalProvider(
      LocalUserRepository provides FakeUserRepository(User("测试用户"))
    ) {
      MyScreen()
    }
  }
  composeRule.onNodeWithText("测试用户").assertExists()
}
```

## 反模式

### ✗ 将 CompositionLocal 用作通用依赖注入
```kotlin
// 糟糕：隐藏依赖，难以测试
val LocalEverything = compositionLocalOf { AppContainer() }

@Composable
fun MyScreen() {
  val container = LocalEverything.current
  val repo = container.userRepo
  val cache = container.cache
}
```

**更好：** 提供具体的 locals 或将依赖作为参数传递。

### ✗ 重复读取 LocalContext
```kotlin
// 低效：每次重组都读取
@Composable
fun MyComposable() {
  val context = LocalContext.current // 重复读取
  // ...
}
```

**更好：** 在 lambda 外部读取一次，或在 remember 中缓存：

```kotlin
@Composable
fun MyComposable() {
  val context = LocalContext.current
  val effect = remember(context) { /* 使用 context */ }
}
```

### ✗ 在 CompositionLocal 中存储可变状态
```kotlin
// 糟糕：状态变化无法正确触发重组
val LocalCounter = compositionLocalOf { mutableStateOf(0) }
```

**更好：** 在父 composable 中存储 State 并提供值，而非 State 本身：

```kotlin
val LocalCount = compositionLocalOf { 0 }

@Composable
fun Parent() {
  var count by remember { mutableStateOf(0) }
  CompositionLocalProvider(LocalCount provides count) {
    Child()
  }
}
```

## 关键要点

1. 子组件读取并依赖更新时用 `compositionLocalOf`
2. 仅对真正静态的值使用 `staticCompositionLocalOf`
3. 除非嵌套很深，否则优先使用参数而非 CompositionLocals
4. 始终提供合理的错误默认值或可空类型
5. 通过 `CompositionLocalProvider` 提供假实现以进行测试
6. CompositionLocals 不是架构的替代品 —— 用于配置和环境数据，而非通用依赖注入
