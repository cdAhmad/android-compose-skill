# Jetpack Compose 导航

参考：`androidx/navigation/navigation-compose/src/commonMain/kotlin/androidx/navigation/compose/`

## 设置

### 基本 NavHost 和 NavController

```kotlin
val navController = rememberNavController()

NavHost(
    navController = navController,
    startDestination = "home" // 使用 Route::class 进行类型安全导航
) {
    composable<Home> {
        HomeScreen(onNavigate = { navController.navigate(Details()) })
    }
}
```

`rememberNavController()` 创建一个在重组中存活的 `NavController`。始终在 `NavHost` 中使用它 — 绝不在 ViewModel 中创建 `NavController`。

## 类型安全导航（Navigation 2.8+）

使用 `@Serializable` 路由类替代字符串路由。这是推荐的模式。

```kotlin
@Serializable
data class Home(val userId: String? = null)

@Serializable
data class Details(val itemId: Int)

NavHost(navController, startDestination = Home()) {
    composable<Home> { backStackEntry ->
        val args = backStackEntry.toRoute<Home>()
        HomeScreen(userId = args.userId)
    }
    composable<Details> { backStackEntry ->
        val args = backStackEntry.toRoute<Details>()
        DetailsScreen(itemId = args.itemId)
    }
}
```

使用 `@Serializable` 在嵌套数据类上序列化复杂类型：

```kotlin
@Serializable
data class User(val id: Int, val name: String)

@Serializable
data class UserProfile(val user: User)

// 导航：
navController.navigate(UserProfile(user = User(1, "Alice")))
```

## 声明目的地

### composable — 屏幕目的地

```kotlin
composable<Route> { backStackEntry ->
    ScreenContent()
}
```

### dialog — 对话框目的地

```kotlin
dialog<Route> { backStackEntry ->
    AlertDialog(...)
}
```

### navigation — 嵌套图（特性模块）

```kotlin
navigation<RootRoute>(startDestination = Home()) {
    composable<Home> { HomeScreen() }
    composable<Details> { DetailsScreen() }
}
```

## 导航

### 导航到目的地

```kotlin
// 类型安全
navController.navigate(Details(itemId = 42))

// 避免：基于字符串的导航
navController.navigate("details/42") // 反模式
```

### 弹出返回栈

```kotlin
navController.popBackStack()

// 带返回值弹出（弹出前保存状态）
navController.previousBackStackEntry?.savedStateHandle?.set("key", value)
navController.popBackStack()

// 在目的地中检索：
val result = navController.currentBackStackEntry?.savedStateHandle?.get<T>("key")
```

### popUpTo — 清空返回栈

```kotlin
// 导航到 Details，从栈中清除 Home
navController.navigate(
    Details(itemId = 42),
    navOptions = navOptions {
        popUpTo(Home::class) { inclusive = false }
    }
)

// inclusive = true：同时移除目标路由
navController.navigate(
    Login(),
    navOptions = navOptions {
        popUpTo(Home::class) { inclusive = true }
    }
)

// launchSingleTop：如果已在栈中则复用现有实例
navController.navigate(
    Details(itemId = 42),
    navOptions = navOptions {
        launchSingleTop = true
    }
)
```

## 参数和返回栈数据

Compose Navigation 使用 `@Serializable` 路由自动处理序列化。

### 传递复杂数据

```kotlin
@Serializable
data class Message(val id: Int, val text: String, val metadata: Metadata)

@Serializable
data class Metadata(val timestamp: Long, val priority: Int)

navController.navigate(Message(1, "Hello", Metadata(System.currentTimeMillis(), 1)))
```

### 通过 SavedStateHandle 传递结果

```kotlin
// 发送结果回传
navController.previousBackStackEntry?.savedStateHandle?.set("result", "success")
navController.popBackStack()

// 在前一屏幕接收
val result = navController.currentBackStackEntry?.savedStateHandle?.get<String>("result")
```

## 嵌套导航图

将相关目的地组织到特性图中。

```kotlin
navigation<FeatureRoot>(startDestination = FeatureHome()) {
    composable<FeatureHome> { FeatureHomeScreen(onNext = { navController.navigate(FeatureDetail()) }) }
    composable<FeatureDetail> { FeatureDetailScreen() }
}
```

好处：作用域 ViewModel、独立的返回栈行为、特性隔离。

## 深层链接

声明深层链接以从 URL 或通知打开应用。

```kotlin
composable<Details>(
    deepLinks = listOf(
        navDeepLink<Details>(
            uriPattern = "https://example.com/details/{itemId}"
        )
    )
) { backStackEntry ->
    val args = backStackEntry.toRoute<Details>()
    DetailsScreen(itemId = args.itemId)
}

// 通过深层链接导航
navController.navigate("https://example.com/details/42")
```

在 `AndroidManifest.xml` 中处理：

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
    </intent-filter>
</activity>
```

## 返回栈管理

### saveState / restoreState

在导航期间保留屏幕状态：

```kotlin
navController.navigate(
    Details(itemId = 42),
    navOptions = navOptions {
        saveState = true
        restoreState = true
    }
)
```

### 检查当前路由

```kotlin
val currentRoute = navController.currentBackStackEntry?.destination?.route
```

### 观察返回栈

```kotlin
val backStackEntry by navController.currentBackStackEntryAsState()
val route = backStackEntry?.destination?.route
```

## 底部导航集成

```kotlin
var selectedItem by remember { mutableStateOf("home") }
val navController = rememberNavController()

Scaffold(
    bottomBar = {
        NavigationBar {
            NavigationBarItem(
                selected = selectedItem == "home",
                onClick = {
                    selectedItem = "home"
                    navController.navigate(Home()) {
                        popUpTo(Home::class) { inclusive = true }
                        launchSingleTop = true
                    }
                },
                icon = { Icon(Icons.Default.Home, null) },
                label = { Text("Home") }
            )
            NavigationBarItem(
                selected = selectedItem == "profile",
                onClick = {
                    selectedItem = "profile"
                    navController.navigate(Profile()) {
                        popUpTo(Home::class) { inclusive = false }
                        launchSingleTop = true
                    }
                },
                icon = { Icon(Icons.Default.Person, null) },
                label = { Text("Profile") }
            )
        }
    }
) {
    NavHost(navController, startDestination = Home()) {
        composable<Home> { HomeScreen() }
        composable<Profile> { ProfileScreen() }
    }
}
```

## 共享元素过渡

```kotlin
NavHost(navController, startDestination = List()) {
    composable<List>(
        sharedTransitionSpec = {
            SharedTransitionLayout()
        }
    ) {
        ListScreen()
    }
    composable<Detail>(
        sharedTransitionSpec = {
            SharedTransitionLayout()
        }
    ) {
        DetailScreen()
    }
}
```

在屏幕中使用：

```kotlin
Image(
    painter = painterResource(id = R.drawable.image),
    contentDescription = null,
    modifier = Modifier.sharedBounds(
        sharedContentState = rememberSharedContentState(key = "image"),
        animatedVisibilityScope = this
    )
)
```

## 使用导航进行 ViewModel 作用域

使用 `hiltViewModel()` 将 ViewModel 作用域到返回栈条目。

```kotlin
composable<Details> { backStackEntry ->
    val viewModel: DetailsViewModel = hiltViewModel()
    DetailsScreen(viewModel = viewModel)
}
```

以此方式作用域的 ViewModel 在配置变化中存活，但在返回栈条目被移除时清除。

## 测试导航

使用 `TestNavHostController` 测试导航行为。

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun navigateToDetails() {
    val navController = TestNavHostController(ApplicationProvider.getApplicationContext())
    navController.navigatorProvider.addNavigator(ComposeNavigator())

    composeTestRule.setContent {
        NavHost(navController, startDestination = Home()) {
            composable<Home> { HomeScreen(onNavigate = { navController.navigate(Details()) }) }
            composable<Details> { DetailsScreen() }
        }
    }

    composeTestRule.onNodeWithTag("detail_button").performClick()
    assertEquals(Details::class.serializer().descriptor.serialName, navController.currentBackStackEntry?.destination?.route)
}
```

## 反模式

### 不要：使用基于字符串的路由
```kotlin
// ❌ 反模式
navController.navigate("details/42")

// ✅ 正确
navController.navigate(Details(itemId = 42))
```

### 不要：在 ViewModel 中创建 NavController
```kotlin
// ❌ 反模式
class MyViewModel : ViewModel() {
    val navController = NavController(context) // 错误！
}

// ✅ 正确
// NavController 存在于 NavHost 中，注入到 composable 中
```

### 不要：在组合中导航
```kotlin
// ❌ 反模式
@Composable
fun MyScreen() {
    if (condition) {
        navController.navigate(Details()) // 每次重组都导航！
    }
}

// ✅ 正确
@Composable
fun MyScreen() {
    LaunchedEffect(condition) {
        if (condition) {
            navController.navigate(Details())
        }
    }
}
```

### 不要：混合导航方式
```kotlin
// ❌ 反模式
navigation<Feature>(startDestination = "home") {
    composable("home") { } // 基于字符串
    composable<Details> { } // 类型安全与字符串混合
}

// ✅ 正确
@Serializable
object FeatureHome

navigation<FeatureRoot>(startDestination = FeatureHome()) {
    composable<FeatureHome> { }
    composable<FeatureDetails> { }
}
```
