# Jetpack Compose 视图组合参考

## Composable 函数命名约定

名称传达意图。始终遵循这些模式。

### 名词（UI 组件）
- PascalCase，描述 composable 显示*什么*
- 用于 UI 控件、屏幕、布局构建块

```kotlin
@Composable
fun Button(...)  // 显示一个按钮

@Composable
fun UserCard(user: User)  // 显示一个用户卡片

@Composable
fun LoginScreen()  // 显示一个登录屏幕

@Composable
fun CheckboxWithLabel(...)  // 显示一个带标签的复选框
```

### 动词（副作用 / Effect）
- PascalCase，描述*发生什么动作*
- 用于只执行副作用而不发射 UI 的 composable

```kotlin
@Composable
fun LaunchedEffect(...)  // 启动一个协程

@Composable
fun DisposableEffect(...)  // 管理资源生命周期

@Composable
fun SideEffect(...)  // 执行一个副作用
```

### 反模式：歧义名称
```kotlin
// ❌ 不清楚这是 UI 组件还是 effect
@Composable
fun HandleLogin(...)

// ✅ 明确
@Composable
fun LoginScreen(...)  // 显示 UI

@Composable
fun PerformLogin(...)  // 副作用函数（如果真的是副作用）
```

## Slot 模式

接受 `@Composable` lambda 参数以创建灵活、可复用的容器。

### 基本 Slot 模式
```kotlin
@Composable
fun Card(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Box(
        modifier = modifier
            .background(Color.White)
            .padding(16.dp)
    ) {
        content()
    }
}

// 使用
Card {
    Text("Hello")
    Image(...)
}
```

### 多个 Slot
```kotlin
@Composable
fun ListItem(
    modifier: Modifier = Modifier,
    icon: @Composable () -> Unit,
    title: @Composable () -> Unit,
    subtitle: @Composable (() -> Unit)? = null,
    trailing: @Composable (() -> Unit)? = null
) {
    Row(modifier = modifier.padding(16.dp)) {
        icon()
        Column(modifier = Modifier.weight(1f)) {
            title()
            subtitle?.invoke()
        }
        trailing?.invoke()
    }
}

// 使用
ListItem(
    icon = { Icon(Icons.Default.Person, "") },
    title = { Text("Alice") },
    subtitle = { Text("Online") },
    trailing = { Badge() }
)
```

**关键原则：** Slot 接受 `@Composable` lambda，而非预先组合的值。这确保组合被推迟且感知作用域。

```kotlin
// ❌ 错误：传递已组合的值
fun CustomLayout(content: String) { ... }

// ✅ 正确：传递组合 lambda
fun CustomLayout(content: @Composable () -> Unit) { ... }
```

来源：`androidx.compose.material3` 中的 Material 3 composable 大量使用了 slot。

## 提取 Composable

知道何时提取以及何时将 composable 放在一起。

### 何时提取
- **在多处复用：** DRY 原则
- **单一职责：** Composable 处理一个关注点
- **更易于测试：** 小、聚焦的单元
- **性能：** 启用跳过和独立重组

```kotlin
// ❌ 之前：上帝 composable
@Composable
fun UserProfile(user: User) {
    Column {
        // 头部
        Box(modifier = Modifier.fillMaxWidth()) {
            Image(user.photo)
            Text(user.name, style = MaterialTheme.typography.headlineSmall)
            IconButton({ /* edit */ }) { Icon(Icons.Default.Edit, "") }
        }

        // 统计
        Row(modifier = Modifier.fillMaxWidth()) {
            StatItem(user.followers, "Followers")
            StatItem(user.following, "Following")
            StatItem(user.posts, "Posts")
        }

        // 简介
        Text(user.bio, style = MaterialTheme.typography.bodyMedium)
    }
}

// ✅ 之后：提取的 composable
@Composable
fun UserProfile(user: User) {
    Column {
        UserProfileHeader(user)
        UserStats(user)
        UserBio(user.bio)
    }
}

@Composable
private fun UserProfileHeader(user: User) { ... }

@Composable
private fun UserStats(user: User) { ... }

@Composable
private fun UserBio(bio: String) { ... }
```

### 何时不提取
- **单次使用：** 提取增加间接性而无益处
- **紧耦合逻辑：** 需要传递很多参数
- **太小：** 单个 `Text()` 或 `Icon()` 不需要提取

```kotlin
// ❌ 过度提取：琐碎包装器
@Composable
private fun UserName(name: String) {
    Text(name, style = MaterialTheme.typography.headlineSmall)
}

// ✅ 如果只使用一次则保持内联
@Composable
fun UserProfile(user: User) {
    Text(user.name, style = MaterialTheme.typography.headlineSmall)
}
```

## 有状态 vs 无状态 Composable

将 composable 结构化为无状态层，并带有可选的有状态包装器。

### 模式：无状态 + 包装器

```kotlin
// ✅ 无状态 composable（可复用、可测试）
@Composable
fun ToggleButton(
    isEnabled: Boolean,
    onToggle: (Boolean) -> Unit,
    text: String
) {
    Button(
        onClick = { onToggle(!isEnabled) },
        colors = ButtonDefaults.buttonColors(
            containerColor = if (isEnabled) Color.Blue else Color.Gray
        )
    ) {
        Text(text)
    }
}

// ✅ 有状态包装器（管理状态，使用无状态子组件）
@Composable
fun StatefulToggleButton(text: String = "Toggle") {
    var isEnabled by remember { mutableStateOf(false) }
    ToggleButton(
        isEnabled = isEnabled,
        onToggle = { isEnabled = it },
        text = text
    )
}

// 使用：根据需求选择
@Composable
fun MyScreen() {
    // 调用方管理状态时使用无状态
    var featureEnabled by remember { mutableStateOf(false) }
    ToggleButton(featureEnabled, { featureEnabled = it }, "Feature")

    // 对简单情况使用有状态包装器
    StatefulToggleButton("Local Toggle")
}
```

**优势：** 调用方无需模拟状态即可测试和复用 `ToggleButton`。`StatefulToggleButton` 为简单情况提供便利。

## Preview 注解

使用 preview 进行快速 UI 开发和回归测试。

### @Preview
单个 composable 的基本 preview。

```kotlin
@Preview
@Composable
fun UserCardPreview() {
    UserCard(user = User(1, "Alice"))
}

// 多个 preview
@Preview(name = "Light")
@Preview(name = "Dark", uiMode = UI_MODE_NIGHT_YES)
@Composable
fun UserCardPreviews() {
    UserCard(user = User(1, "Alice"))
}
```

### @PreviewLightDark
自动生成亮色和暗色主题 preview。

```kotlin
@PreviewLightDark
@Composable
fun UserCardPreview() {
    MyTheme {
        UserCard(user = User(1, "Alice"))
    }
}
```

### @PreviewFontScale
显示 composable 如何响应不同字体大小。

```kotlin
@Preview(fontScale = 0.8f, name = "Small Fonts")
@Preview(fontScale = 1f, name = "Normal Fonts")
@Preview(fontScale = 1.2f, name = "Large Fonts")
@Composable
fun TextPreview() {
    Text("This is text")
}
```

### @PreviewScreenSizes
测试多种屏幕尺寸。

```kotlin
@Preview(device = Devices.PHONE, name = "Phone")
@Preview(device = Devices.TABLET, name = "Tablet")
@Preview(device = Devices.FOLDABLE, name = "Foldable")
@Composable
fun ResponsiveLayoutPreview() {
    ResponsiveLayout()
}
```

来源：`androidx.compose.ui.tooling.preview`

## CompositionLocal

无需通过层级逐层传递即可提供隐式参数。

### 何时使用
- **主题化：** 隐式传递 `Colors`、`Typography`
- **导航：** 从 composable 树深处访问
- **区域设置/字符串：** 避免通过每个 composable 传递

```kotlin
// ✅ 在顶层定义
val LocalUser = staticCompositionLocalOf<User?> { null }

@Composable
fun App(user: User) {
    CompositionLocalProvider(LocalUser provides user) {
        MainContent()
    }
}

// ✅ 深入树中访问，无需通过每个 composable 传递
@Composable
fun UserGreeting() {
    val user = LocalUser.current
    Text("Hello, ${user?.name}")
}
```

### 何时不使用
- **配置参数：** 如果只深 1-2 层，直接传递
- **频繁变化的值：** 可能导致不必要的重组
- **依赖项：** 在 ViewModel 级别使用依赖注入

```kotlin
// ❌ 过度使用：应直接传递 `title`
val LocalTitle = staticCompositionLocalOf<String> { "" }

@Composable
fun Parent() {
    CompositionLocalProvider(LocalTitle provides "My Title") {
        Child()
    }
}

// ✅ 直接传递即可
@Composable
fun Parent(title: String) {
    Child(title = title)
}

@Composable
fun Child(title: String) { ... }
```

**类型（关键区别在于重组范围）：**
- `staticCompositionLocalOf`：值变化时，`CompositionLocalProvider` 下方的**整个子树**失效并重组。用于在组合期间真正从不变化的值（主题、间距、DI 提供的依赖）。
- `compositionLocalOf`：值变化时，只有**实际读取** `.current` 的 composable 失效。用于可能在组合期间变化的值（用户会话、区域设置、滚动状态）。

## Composable 返回值

**永远不要从 composable 返回值。** 使用回调代替。

```kotlin
// ❌ 错误：composable 不返回值
@Composable
fun UserInput(): String {
    var text by remember { mutableStateOf("") }
    return text  // 不能这样做
}

// ✅ 正确：回调模式
@Composable
fun UserInput(onUserInput: (String) -> Unit) {
    var text by remember { mutableStateOf("") }
    TextField(
        value = text,
        onValueChange = {
            text = it
            onUserInput(it)  // 通知父组件
        }
    )
}

// 使用
@Composable
fun FormScreen() {
    UserInput(onUserInput = { input -> /* handle */ })
}
```

**原理：** Composable 在组合期间执行，这发生在不可预测的时间，可能被跳过或重新排序。

## Screen 级别 Composable

将 screen 结构化为纯 composable 之上的薄 ViewModel 集成层。

### 推荐模式
```kotlin
// ✅ Screen composable：连接 ViewModel
@Composable
fun UserDetailsScreen(
    viewModel: UserDetailsViewModel = hiltViewModel(),
    userId: String
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    UserDetailsContent(
        uiState = uiState,
        onRetry = { viewModel.loadUser(userId) }
    )
}

// ✅ Content composable：纯的（可测试、可复用）
@Composable
private fun UserDetailsContent(
    uiState: UiState,
    onRetry: () -> Unit
) {
    when (uiState) {
        is UiState.Loading -> LoadingUI()
        is UiState.Success -> SuccessUI(uiState.user)
        is UiState.Error -> ErrorUI(uiState.message, onRetry)
    }
}

// ✅ 用于 preview/测试的 composable
@Preview
@Composable
private fun UserDetailsContentPreview() {
    UserDetailsContent(
        uiState = UiState.Success(User(1, "Alice")),
        onRetry = {}
    )
}
```

**好处：**
- Public screen composable 集成 ViewModel
- Private content composable 是纯的、可测试的、可 preview 的
- 清晰的分离：UI 逻辑（public）vs 渲染（private）

**反模式：** 将 ViewModel 传递给子 composable。只保持在 screen 级别。

```kotlin
// ❌ 将子组件与 ViewModel 耦合
@Composable
fun UserCard(viewModel: UserViewModel) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    Text(user.name)
}

// ✅ 只传递数据
@Composable
fun UserCard(user: User) {
    Text(user.name)
}
```

## 可复用性指南

设计 composable 使其可配置但不过度参数化。

### 通过参数配置
```kotlin
// ✅ 暴露变化的部分，隐藏不变的部分
@Composable
fun Card(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    // 内部：固定样式、padding 等
    Box(
        modifier = modifier
            .background(Color.White)
            .padding(16.dp)
            .clip(RoundedCornerShape(8.dp))
    ) {
        content()
    }
}
```

### 避免参数爆炸
```kotlin
// ❌ 太多参数，难以使用
@Composable
fun Button(
    text: String,
    textColor: Color,
    backgroundColor: Color,
    cornerRadius: Dp,
    padding: PaddingValues,
    elevation: Dp,
    ...
)

// ✅ 合理的默认值、样式对象
@Composable
fun Button(
    text: String,
    modifier: Modifier = Modifier,
    style: ButtonStyle = ButtonStyle.Primary,
    onClick: () -> Unit
) { ... }

// 或者：使用内置样式的 Material composable
@Composable
fun Button(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    ...
) { ... }  // Material3 Button 有合理的默认值
```

## 常见反模式

### 上帝 Composable
```kotlin
// ❌ 做得太多
@Composable
fun Dashboard() {
    // 头部
    Box { /* 20 行 */ }

    // 列表
    LazyColumn {
        items(items) { /* 15 行 */ }
    }

    // 底部
    Box { /* 15 行 */ }

    // 对话框、副作用、状态管理...
}

// ✅ 提取并委托
@Composable
fun Dashboard() {
    Column {
        DashboardHeader()
        DashboardContent()
        DashboardFooter()
    }
}
```

### 深层嵌套
```kotlin
// ❌ 难以阅读和调试
@Composable
fun LoginScreen() {
    Box { Column { Row { Card { Box { Text { ... } } } } } }
}

// ✅ 中间变量和提取
@Composable
fun LoginScreen() {
    val form = loginFormState()
    Column {
        LoginForm(form)
        LoginButton(form)
    }
}
```

### 将 ViewModel 传递给子组件
```kotlin
// ❌ 违反组合边界
@Composable
fun ParentScreen(viewModel: ParentViewModel) {
    ChildCard(viewModel = viewModel)  // 不要这样做
}

// ✅ 提取数据，传递给子组件
@Composable
fun ParentScreen(viewModel: ParentViewModel) {
    val data by viewModel.data.collectAsStateWithLifecycle()
    ChildCard(data = data)
}
```

---

**来源参考：** `androidx.compose.material3`、`androidx.compose.ui.tooling.preview`、`androidx.compose.runtime.CompositionLocal`

## 设计到 Composable 分解

将视觉设计（Figma 画框、截图或规范）翻译成 composable 树的系统性 5 步过程：

**步骤 1：识别根布局结构**
- 全屏 Scaffold？（TopAppBar + 内容 + 底栏 + FAB）
- 可滚动内容？（LazyColumn vs 带 verticalScroll 的 Column）
- 标签布局？（TabRow + HorizontalPager）
- 对话框或底栏？

**步骤 2：自上而下分解为视觉区块**
- 识别主要水平区块（头部、内容区、底部）
- 在每个区块内，识别水平分组（图标 + 文本行、卡片网格）
- 这镜像 DCGen 分而治之的方法：先水平分割，再垂直分割

**步骤 3：对每个区块，识别布局类型**
- 垂直堆叠且等间距的条目 -> 带 `Arrangement.spacedBy()` 的 `Column`
- 并排的条目 -> 带 weight 或固定尺寸的 `Row`
- 重叠的条目 -> 带 alignment modifier 的 `Box`
- 卡片网格 -> `LazyGrid` 或 `FlowRow`
- 可滚动的条目列表 -> `LazyColumn`

**步骤 4：提取视觉属性并映射到主题**
- 背景色 -> `MaterialTheme.colorScheme.*`
- 排版 -> `MaterialTheme.typography.*`（headlineLarge、bodyMedium 等）
- 间距 -> 4dp/8dp 网格增量，使用 `Arrangement.spacedBy()` 和 `Modifier.padding()`
- 圆角 -> `MaterialTheme.shapes.*`
- 阴影 -> 带 `tonalElevation` 的 `Card` 或 `Surface`

**步骤 5：识别交互元素**
- 按钮、文本框、切换、复选框 -> 映射到 Material 3 组件
- 自定义可点击区域 -> 带 `role = Role.Button` 的 `Modifier.clickable`
- 确保 48dp 最小触摸目标

## Screen 结构模式

标准 screen 模式将 ViewModel 集成与 UI 分离：

```kotlin
@Composable
fun ConversationScreen(
    viewModel: ConversationViewModel = hiltViewModel(),
    onNavigateToDetail: (String) -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    ConversationContent(
        uiState = uiState,
        onAction = viewModel::onAction,
        onNavigateToDetail = onNavigateToDetail
    )
}

@Composable
private fun ConversationContent(
    uiState: ConversationUiState,
    onAction: (ConversationAction) -> Unit,
    onNavigateToDetail: (String) -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Conversations") })
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { onAction(ConversationAction.Create) }) {
                Icon(Icons.Default.Add, contentDescription = "New conversation")
            }
        }
    ) { innerPadding ->
        // 必须使用 innerPadding -- 忽略它会导致内容重叠
        when (val state = uiState) {
            is ConversationUiState.Loading -> {
                Box(Modifier.fillMaxSize().padding(innerPadding), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is ConversationUiState.Success -> {
                LazyColumn(modifier = Modifier.padding(innerPadding)) {
                    items(state.conversations, key = { it.id }) { conversation ->
                        ConversationRow(
                            conversation = conversation,
                            onClick = { onNavigateToDetail(conversation.id) }
                        )
                    }
                }
            }
            is ConversationUiState.Error -> {
                ErrorContent(state.message, modifier = Modifier.padding(innerPadding))
            }
        }
    }
}
```

关键模式：ViewModel 在 screen 级别，纯 content composable 在其下方。Content composable 接收 state + 回调，绝不接收 ViewModel。这使其可 preview 和可测试。

## 组合 Preview 注解

一次定义，到处使用：

```kotlin
@Preview(name = "Light", uiMode = Configuration.UI_MODE_NIGHT_NO)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Preview(name = "Large Font", fontScale = 1.5f)
@Preview(name = "Small Device", device = "spec:width=320dp,height=640dp,dpi=320")
@Preview(name = "Tablet", device = Devices.TABLET)
@Preview(name = "Foldable", device = Devices.FOLDABLE)
@Preview(name = "RTL", locale = "ar")
annotation class ComponentPreviews
```

应用到每个提取的 composable：
```kotlin
@ComponentPreviews
@Composable
private fun ConversationRowPreview() {
    AppTheme {
        ConversationRow(
            conversation = previewConversation(),
            onClick = {}
        )
    }
}
```

对于数据驱动的 preview，使用 `PreviewParameterProvider`：
```kotlin
class ConversationPreviewProvider : PreviewParameterProvider<Conversation> {
    override val values = sequenceOf(
        Conversation(id = "1", title = "Short title", unreadCount = 0),
        Conversation(id = "2", title = "Very long conversation title that might wrap", unreadCount = 99),
        Conversation(id = "3", title = "", unreadCount = 0), // 空标题边界情况
    )
}

@ComponentPreviews
@Composable
private fun ConversationRowPreview(
    @PreviewParameter(ConversationPreviewProvider::class) conversation: Conversation
) {
    AppTheme { ConversationRow(conversation = conversation, onClick = {}) }
}
```

设备特定的 preview（`Devices.TABLET`）需要来自 `androidx.compose.ui.tooling.preview` 的 Android `@Preview` 注解。

## 自适应布局

使用 `WindowSizeClass` 为不同屏幕尺寸调整布局：

```kotlin
@Composable
fun AdaptiveScreen(windowSizeClass: WindowSizeClass) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // 手机：单列
            SinglePaneLayout()
        }
        WindowWidthSizeClass.Medium -> {
            // 小平板：双面板
            TwoPaneLayout()
        }
        WindowWidthSizeClass.Expanded -> {
            // 大平板/桌面：列表-详情
            ListDetailLayout()
        }
    }
}
```

对于导航，使用 `NavigationSuiteScaffold`，它自动在底部导航（compact）、侧边栏（medium）和抽屉（expanded）之间切换。
