# 原子设计系统参考

在 Jetpack Compose 中构建可复用的、层级化的组件系统。
基于 Brad Frost 的原子设计方法论，映射到 Compose 原语。

---

## 1. 五级层级映射到 Compose

| 层级 | Compose 等价物 | 示例 |
|-------|-------------------|----------|
| **Tokens** | `MaterialTheme.colorScheme.*`、`MaterialTheme.typography.*`、自定义 `CompositionLocal` token（间距、阴影、品牌色） | `AppTheme.spacing.medium`、`AppTheme.colors.brandPrimary` |
| **Atoms** | 单一职责、slot API、modifier 参数的单用途 composable。包裹 M3 或自定义构建。 | `AppButton`、`AppTextField`、`AppAvatar`、`AppIcon` |
| **Molecules** | 将 2+ 个 atom 组合成功能单元的 composable | `SearchBar`（图标 + 文本框）、`MovieCard`（图片 + 文本）、`UserListItem` |
| **Organisms** | 将 molecule 组合成 UI 区域的屏幕区块 | `MovieCatalogRow`（标题 + MovieCard 的 LazyRow）、`NavigationDrawerWithContent` |
| **Templates** | 定义内容区域但不绑定数据的屏幕布局 — `Scaffold` + slot 组合 | `MainScreenTemplate(topBar, content, bottomBar)`、`DetailScreenTemplate(hero, body, actions)` |

**依赖规则：** 每一层只依赖其下方的层。Organism 不应使用
原始 `Text()` — 它应使用 atom。Molecule 不应硬编码颜色 — 它应通过
`MaterialTheme` 或自定义 `CompositionLocal` 使用 token。

```
Template
  └── Organism
        └── Molecule
              └── Atom
                    └── Token (MaterialTheme / CompositionLocal)
```

---

## 2. Token 层

Token 是基础。每个视觉属性（颜色、排版、间距、形状、动效）
都应来自 token — 绝不要在 composable 体内硬编码。

### M3 token（直接使用）

这些已由 `MaterialTheme` 提供：

- `MaterialTheme.colorScheme` — primary、secondary、surface、error 等
- `MaterialTheme.typography` — displayLarge 到 labelSmall
- `MaterialTheme.shapes` — extraSmall 到 extraLarge
- `MaterialTheme.motionScheme` — `defaultSpatialSpec()`、`defaultEffectsSpec()`

### 应用级自定义 token

当 M3 无法满足需求时创建。使用 `CompositionLocal` + 包裹主题。

**间距尺度：**
```kotlin
object AppSpacing {
    val xxs = 2.dp
    val xs = 4.dp
    val sm = 8.dp
    val md = 16.dp
    val lg = 24.dp
    val xl = 32.dp
}
val LocalAppSpacing = staticCompositionLocalOf { AppSpacing }
```

**品牌色（超出 M3 colorScheme）：**
```kotlin
data class AppBrandColors(
    val accent: Color,
    val onAccent: Color,
    val surface: Color,
)
val LocalAppBrandColors = staticCompositionLocalOf {
    AppBrandColors(
        accent = Color.Unspecified,
        onAccent = Color.Unspecified,
        surface = Color.Unspecified,
    )
}
```

**访问模式 — 包裹在 `AppTheme` 中：**
```kotlin
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(
        LocalAppSpacing provides AppSpacing,
        LocalAppBrandColors provides AppBrandColors(
            accent = Color(0xFF1A73E8),
            onAccent = Color.White,
            surface = Color(0xFFF5F5F5),
        )
    ) {
        MaterialTheme(
            colorScheme = /* 你的 color scheme */,
            typography = /* 你的 typography */,
            shapes = /* 你的 shapes */,
        ) {
            content()
        }
    }
}

// 在树中任意位置使用：
val spacing = LocalAppSpacing.current
val brandColors = LocalAppBrandColors.current
```

**何时创建自定义 token vs. 直接使用 M3：**
- M3 已覆盖 → 直接使用 `MaterialTheme.*`
- 应用专属概念（品牌强调色、间距尺度、阴影尺度） → 自定义 `CompositionLocal`
- 单个组件中需要的一次性值 → 不是 token，只是局部常量

---

## 3. Atom 模式

Atom 是最小的可复用 UI 单元。每个 atom 必须满足 **atom 契约**。

### Atom 契约

每个 atom（渲染 UI 的 public composable）必须满足：

1. **`modifier: Modifier = Modifier` 参数** — 调用方控制布局
2. **可变内容的 Slot API** — `@Composable () -> Unit` 或带作用域的如 `@Composable RowScope.() -> Unit`
3. **基于 Token 的样式** — 无硬编码 `Color(0xFF...)`、`14.sp`、`FontWeight.Bold`
4. **合理的默认值** — 无需配置即可工作
5. **Preview composable** — 用于视觉验证的 `@Preview` 函数

### 两种 atom 类型

**1. M3 包裹 atom** — 用品牌默认值包裹 M3 组件：

```kotlin
@Composable
fun AppButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    content: @Composable RowScope.() -> Unit,
) {
    Button(
        onClick = onClick,
        modifier = modifier,
        enabled = enabled,
        colors = ButtonDefaults.buttonColors(
            containerColor = LocalAppBrandColors.current.accent,
            contentColor = LocalAppBrandColors.current.onAccent,
        ),
        content = content,
    )
}
```

**2. 自定义 atom** — 无 M3 等价物时：

```kotlin
@Composable
fun AppAvatar(
    imageUrl: String,
    size: AvatarSize = AvatarSize.Medium,
    modifier: Modifier = Modifier,
    contentDescription: String? = null,
) {
    AsyncImage(
        model = imageUrl,
        contentDescription = contentDescription,
        contentScale = ContentScale.Crop,
        modifier = modifier
            .size(size.dp)
            .clip(CircleShape)
    )
}

enum class AvatarSize(val dp: Dp) {
    Small(24.dp), Medium(40.dp), Large(56.dp)
}
```

### 命名规则

按组件**是什么**命名，而非在哪里使用。

| 不好 | 好 | 原因 |
|-----|------|-----|
| `ButtonWithBoldCTA` | `AppButton` | 粗体是样式变体，不是组件 |
| `RedBorderCard` | `HighlightCard` 或 `AppCard` | 按视觉外观命名，而非功能 |
| `HomeMovieCard` | `MovieCard` | 按屏幕命名，不可复用 |
| `ButtonForSettings` | `AppButton` | 按上下文命名，而非功能 |

---

## 4. Molecule、Organism 和 Template 模式

### Molecule — 组合 2+ 个 atom

Molecule 将 atom 组合成功能单元。它接受数据和回调，不接受 ViewModel。

```kotlin
@Composable
fun MovieCard(
    movie: Movie,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    AppCard(onClick = onClick, modifier = modifier) {
        AppImage(url = movie.posterUrl, contentDescription = movie.title)
        AppText(text = movie.title, style = MaterialTheme.typography.titleSmall)
        AppText(text = movie.year.toString(), style = MaterialTheme.typography.bodySmall)
    }
}
```

### Organism — 将 molecule 组合成 UI 区域

Organism 是屏幕区块。它仍然接受数据作为参数 — 绝不直接读取 ViewModel。

```kotlin
@Composable
fun MovieCatalogRow(
    title: String,
    movies: List<Movie>,
    onMovieClick: (Movie) -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(modifier = modifier) {
        AppText(text = title, style = MaterialTheme.typography.headlineSmall)
        LazyRow(
            horizontalArrangement = Arrangement.spacedBy(LocalAppSpacing.current.sm)
        ) {
            items(movies, key = { it.id }) { movie ->
                MovieCard(movie = movie, onClick = { onMovieClick(movie) })
            }
        }
    }
}
```

### Template — 通过 slot 组合定义屏幕布局

Template 定义内容放在哪里，而非内容是什么。它接受 slot 参数，不接受数据。

```kotlin
@Composable
fun CatalogScreenTemplate(
    topBar: @Composable () -> Unit,
    hero: @Composable () -> Unit,
    sections: @Composable () -> Unit,
    modifier: Modifier = Modifier,
) {
    Scaffold(topBar = topBar, modifier = modifier) { padding ->
        LazyColumn(contentPadding = padding) {
            item { hero() }
            item { sections() }
        }
    }
}
```

### 层级总结

| 层级 | 接受 | 组合 | ViewModel? |
|-------|---------|----------|-----------|
| Atom | 原语、slot、modifier | M3 组件或原始 Compose | 否 |
| Molecule | 数据类、回调、modifier | Atom | 否 |
| Organism | 数据、回调、modifier | Molecule + atom | 否 |
| Template | 仅 slot、modifier | Scaffold + 布局 | 否 |
| Screen | ViewModel | Template + organism + molecule | 是 — 这是唯一接触 ViewModel 的层级 |

---

## 5. "Ask" 提示

当 skill 检测到构建组件的意图（用户要求 "build a card"、"create a button"、
"implement this component"），**在搭建代码之前**，询问：

> "这看起来是一个 **[molecule/organism]**。是否需要我同时搭建它所需的 **[下层] atom**，
> 还是你的代码库中已经有了？"

开发者可以回答：

| 回答 | Skill 行为 |
|--------|---------------|
| "是的，全部搭建" | 从 token 层开始创建 — 定义间距/颜色 token、atom，然后是请求的组件 |
| "只构建卡片" | 使用原子原则（slot、modifier、token）构建请求的组件，但不创建下层 atom |
| "我们已有 AppButton、AppImage" | 复用这些 atom，只构建新的 molecule/organism |

**无论回答如何，skill 始终应用原子原则。** 问题只关乎
是否搭建下层。每个组件都获得：
- `modifier: Modifier = Modifier`
- 适当的 Slot API
- 基于 Token 的样式（无硬编码值）
- 合理的默认值

---

## 6. 反模式

| 反模式 | 错误原因 | 修复 |
|-------------|---------------|-----|
| composable 体内使用 `Color(0xFF1A73E8)` | 硬编码颜色 — 无法换主题，不支持暗色模式 | 使用 `MaterialTheme.colorScheme.*` 或应用品牌 token |
| `fontSize = 14.sp`、`fontWeight = FontWeight.Bold` | 硬编码排版破坏一致性 | 使用 `MaterialTheme.typography.*` |
| `Modifier.padding(16.dp)` 无间距 token | 魔法数字间距 — 应用各处不一致 | 使用 `LocalAppSpacing.current.md`（或定义间距尺度） |
| 名为 `ButtonForSettings` / `CardWithRedBorder` 的 composable | 按上下文命名，而非功能 — 不可复用 | 按它**是什么**命名：`AppButton`、`HighlightCard` |
| Public composable 无 `modifier` 参数 | 调用方无法控制布局 | 添加 `modifier: Modifier = Modifier`，传递给根元素 |
| 渲染 UI 的 composable 无 slot 参数 | 内容硬编码，不可组合 | 为可变内容添加 slot API |
| Organism 中使用原始 `Text()` / `Button()` / `Icon()` | 跳过原子层级 — 丢失主题和一致性 | 使用应用级 atom 包裹器 |
| Organism 直接读取 ViewModel | 将 UI 与数据层耦合 — 不可复用、无法预览 | 接受数据和回调作为参数；让 screen 调用 ViewModel |
| 承担 3–4 个以上职责的 Molecule | 一个组件中塞太多 — 难以复用局部 | 分解为更小的 molecule 或提取 atom |
