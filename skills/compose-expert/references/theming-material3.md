# Material 3 主题参考

## MaterialTheme 基础

`MaterialTheme` 是 Compose Material 3 中设计 token 的根提供者。它在整个应用中建立 `colorScheme`、`typography` 和 `shapes`。

```kotlin
@Composable
fun MyApp() {
    MaterialTheme(
        colorScheme = lightColorScheme(),
        typography = Typography(),
        shapes = Shapes()
    ) {
        // 所有后代通过 MaterialTheme 访问 token
        Scaffold {
            Text("Uses MaterialTheme.typography.bodyLarge")
        }
    }
}
```

**来源**：`androidx/compose/material3/MaterialTheme.kt`

---

## ColorScheme — 亮色与暗色

`ColorScheme` 捆绑 29+ 语义化颜色 token（primary、secondary、error、surface 等）。

### 默认亮色/暗色方案

```kotlin
// 亮色（默认）
val lightColors = lightColorScheme(
    primary = Color(0xFF6200EE),
    secondary = Color(0xFF03DAC6)
)

// 暗色
val darkColors = darkColorScheme(
    primary = Color(0xFFBB86FC),
    secondary = Color(0xFF03DAC6)
)

MaterialTheme(colorScheme = if (isDark) darkColors else lightColors) { ... }
```

### 动态取色（Material You）

Android 12+ 支持从壁纸提取颜色。调用前检查 `Build.VERSION.SDK_INT`：

```kotlin
val colorScheme = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    val context = LocalContext.current
    if (isDark) {
        dynamicDarkColorScheme(context)
    } else {
        dynamicLightColorScheme(context)
    }
} else {
    if (isDark) darkColorScheme() else lightColorScheme()
}

MaterialTheme(colorScheme = colorScheme) { ... }
```

这需要 `android:READ_MEDIA_IMAGES` 或 context 访问权限。`dynamicColorScheme` API 位于 `androidx.compose.material3`。

---

## Typography — 自定义字体尺度

`Typography` 定义 `displayLarge`、`headlineSmall`、`bodyLarge`、`labelSmall` 等。

### 使用 Google Fonts

```kotlin
val Poppins = FontFamily(
    Font(R.font.poppins_regular, weight = FontWeight.Normal),
    Font(R.font.poppins_bold, weight = FontWeight.Bold),
    Font(R.font.poppins_italic, weight = FontWeight.Normal, style = FontStyle.Italic)
)

val customTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = Poppins,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    ),
    bodyMedium = TextStyle(
        fontFamily = Poppins,
        fontSize = 14.sp,
        lineHeight = 20.sp
    )
)

MaterialTheme(typography = customTypography) { ... }
```

所有 M3 字体样式遵循 15 级尺度。部分覆盖保持未指定样式的默认值。

---

## Shapes — 圆角自定义

`Shapes` 定义 `extraSmall`、`small`、`medium`、`large`、`extraLarge` 圆角半径。

```kotlin
val customShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small = RoundedCornerShape(8.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(28.dp)
)

MaterialTheme(shapes = customShapes) { ... }

// 在组件中使用
Button(
    modifier = Modifier.clip(MaterialTheme.shapes.large)
) { ... }
```

组件通过 `Surface` 和 `Card` 自动使用主题形状。

---

## 暗色主题

### isSystemInDarkTheme()

检查系统暗色模式设置：

```kotlin
@Composable
fun MyApp() {
    val isDark = isSystemInDarkTheme()
    MaterialTheme(colorScheme = if (isDark) darkColorScheme() else lightColorScheme()) {
        // 内容
    }
}
```

### 手动切换并持久化

用于用户可选的暗色模式：

```kotlin
val darkModeState = rememberSaveable { mutableStateOf(isSystemInDarkTheme()) }

MaterialTheme(colorScheme = if (darkModeState.value) darkColorScheme() else lightColorScheme()) {
    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = { darkModeState.value = !darkModeState.value }) {
                Icon(Icons.Default.Settings, "Toggle theme")
            }
        }
    ) {
        // 内容
    }
}
```

通过 DataStore 或 SharedPreferences 持久化选择。

---

## 组件级样式

使用 `MaterialTheme` token 作为颜色，而非硬编码值：

```kotlin
// 应做
Text(
    text = "Hello",
    color = MaterialTheme.colorScheme.onSurface,
    style = MaterialTheme.typography.bodyLarge
)

// 不应做
Text(text = "Hello", color = Color.Black, fontSize = 14.sp)
```

常用 token：
- `primary`、`secondary`、`tertiary` — 强调色
- `surface`、`surfaceVariant` — 容器背景
- `onPrimary`、`onSurface`、`onError` — 彩色背景上的文字/内容
- `error`、`errorContainer` — 错误状态

---

## Surface vs Box

**Surface**：具有阴影、来自 `colorScheme.surface` 的背景、尊重主题的样式化容器。

```kotlin
Surface(
    color = MaterialTheme.colorScheme.surface,
    modifier = Modifier.fillMaxWidth()
) {
    // 有阴影、阴影效果
    Text("Themed container")
}
```

**Box**：普通容器，无主题假设。

```kotlin
Box(
    modifier = Modifier
        .fillMaxWidth()
        .background(MaterialTheme.colorScheme.surface)
) {
    // 手动样式
    Text("Manual background")
}
```

对语义容器（卡片、对话框）使用 `Surface`。对布局分组使用 `Box`。

---

## Scaffold — 布局集成

`Scaffold` 组合 `topBar`、`floatingActionButton`、`snackbarHost` 和内容，并带有适当的内边距：

```kotlin
Scaffold(
    topBar = {
        TopAppBar(
            title = { Text("My App") },
            colors = TopAppBarDefaults.topAppBarColors(
                containerColor = MaterialTheme.colorScheme.primaryContainer
            )
        )
    },
    floatingActionButton = {
        FloatingActionButton(
            onClick = { /* ... */ },
            containerColor = MaterialTheme.colorScheme.tertiary
        ) {
            Icon(Icons.Default.Add, "Add item")
        }
    },
    snackbarHost = { SnackbarHost(it) }
) { innerPadding ->
    LazyColumn(modifier = Modifier.padding(innerPadding)) {
        // 内容尊重安全区域
    }
}
```

`Scaffold` 自动处理 inset 和间距。不要嵌套 `Scaffold` 组件。

---

## 使用 CompositionLocal 扩展主题

添加自定义设计 token：

```kotlin
data class AppColors(
    val brandPrimary: Color = Color(0xFF6200EE),
    val brandSecondary: Color = Color(0xFF03DAC6),
    val neutral: Color = Color.Gray
)

val LocalAppColors = compositionLocalOf { AppColors() }

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    val appColors = AppColors()
    CompositionLocalProvider(LocalAppColors provides appColors) {
        MaterialTheme {
            content()
        }
    }
}

// 使用自定义 token
@Composable
fun MyButton() {
    Button(
        colors = ButtonDefaults.buttonColors(
            containerColor = LocalAppColors.current.brandPrimary
        )
    ) { }
}
```

---

## 反模式

### 硬编码颜色
```kotlin
// 不应做
Text("Hello", color = Color(0xFF000000))

// 应做
Text("Hello", color = MaterialTheme.colorScheme.onSurface)
```

### 在可复用组件中忽略主题
```kotlin
// 不应做
fun MyCard(content: @Composable () -> Unit) {
    Box(modifier = Modifier.background(Color.White))
}

// 应做
fun MyCard(content: @Composable () -> Unit) {
    Surface(
        color = MaterialTheme.colorScheme.surface,
        shape = MaterialTheme.shapes.medium
    ) {
        content()
    }
}
```

### 混合 Material 2 和 Material 3
不要同时导入 `androidx.compose.material` 和 `androidx.compose.material3`。新项目选择 M3。

### 未提供所有主题参数
部分 `MaterialTheme` 调用可能使后代使用默认值：

```kotlin
// 如果 colorScheme 因语言而异则不安全
MaterialTheme(typography = customTypography) { ... }

// 更好
MaterialTheme(
    colorScheme = currentColorScheme,
    typography = currentTypography,
    shapes = currentShapes
) { ... }
```

---

## 资源

- **Material 3 Tokens**: https://m3.material.io/
- **Compose Material3 文档**: https://developer.android.com/develop/ui/compose/designsystems/material3
- **Dynamic Color**: 需要 `androidx.compose.material3:material3` >= 1.1.0 和 Android 12+
