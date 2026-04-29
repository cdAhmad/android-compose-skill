# Design-to-Compose 翻译参考

将视觉设计（Figma 模型、截图、线框图）翻译为生产级 Compose 代码。本指南提供系统性的分解算法、属性映射表以及能够在第一次就产出干净、感知主题的 composable 的模式。

---

## 1. Composable 分解算法

一种分而治之的方法，用于将任何视觉设计分解为 composable 函数。自上而下、由外而内地工作。

### 步骤 1：识别根布局结构

首先看完整屏幕。最外层的结构模式是什么？

| 视觉模式 | Compose 根组件 |
|---|---|
| 顶部栏 + 内容 + 底部栏 | `Scaffold` |
| 可滚动垂直内容 | `Column` + `verticalScroll()` 或 `LazyColumn` |
| 标签页分区 | `Scaffold` + `TabRow` + `HorizontalPager` |
| 全出血背景叠加层 | `Box` |
| 侧边抽屉 + 内容 | `ModalNavigationDrawer` + `Scaffold` |
| 内容上方的底部浮层 | `ModalBottomSheet` 或 `Scaffold` + `BottomSheetScaffold` |

### 步骤 2：自上而下分解为视觉区块

从上到下扫描设计。在视觉上不同的区块之间画水平线。每个区块成为一个 composable 或父布局内的代码块。

```
+---------------------------+
|  Top App Bar              |  -> TopAppBar()
+---------------------------+
|  Hero Image               |  -> HeroSection()
+---------------------------+
|  Title + Subtitle         |  -> HeaderSection()
+---------------------------+
|  Horizontal card list     |  -> FeaturedCardsRow()
+---------------------------+
|  Vertical item list       |  -> ItemList()
+---------------------------+
|  Bottom navigation        |  -> NavigationBar()
+---------------------------+
```

**应做：** 按用途命名区块（`FeaturedCardsRow`），而不是按布局（`HorizontalScrollRow`）。

**不应做：** 为每个 Figma 画框创建 composable。尽可能扁平化。

### 步骤 3：为每个区块识别布局类型

```
内容是垂直堆叠的吗？
  └─ 是 -> Column
       └─ 列表是动态/长的吗？ -> LazyColumn

内容是水平排列的吗？
  └─ 是 -> Row
       └─ 它会滚动吗？ -> LazyRow
       └─ 它会换行吗？ -> FlowRow

内容是重叠/层叠的吗？
  └─ 是 -> Box

它是网格吗？
  └─ 固定列数 -> LazyVerticalGrid
  └─ 固定条目大小 -> LazyVerticalStaggeredGrid
  └─ 换行的 chip/tag -> FlowRow
```

**布局选择决策树：**

```
                    ┌─ Overlapping layers? ──→ Box
                    │
  Visual section ───┼─ Single axis? ──→ Vertical? ──→ Column / LazyColumn
                    │                  └─ Horizontal? ──→ Row / LazyRow
                    │
                    └─ Grid / wrap? ──→ Fixed columns? ──→ LazyVerticalGrid
                                       └─ Flowing tags? ──→ FlowRow
```

### 步骤 4：提取视觉属性

对于每个元素，从设计中读取以下内容：

- **Colors** — 映射到 `MaterialTheme.colorScheme.*` token，而非十六进制值
- **Typography** — 映射到 `MaterialTheme.typography.*` 文本样式
- **Spacing** — 以 dp 为单位的 padding 和间隙，映射到主题间距 token
- **Elevation** — 阴影深度，映射到 `tonalElevation` 或 `shadowElevation`
- **Corner radius** — 映射到 `MaterialTheme.shapes.*`

### 步骤 5：识别交互元素并映射到 M3 组件

| 视觉元素 | Compose M3 组件 |
|---|---|
| 圆角矩形文本 + 点击 | `Button` / `OutlinedButton` / `TextButton` |
| 带图片、标题、副标题的卡片 | `Card` / `ElevatedCard` / `OutlinedCard` |
| 文本输入框 | `TextField` / `OutlinedTextField` |
| 切换开关 | `Switch` |
| 复选框 | `Checkbox` / `TriStateCheckbox` |
| Chip / 标签 | `FilterChip` / `AssistChip` / `InputChip` / `SuggestionChip` |
| 浮动操作按钮 | `FloatingActionButton` / `ExtendedFloatingActionButton` |
| 底部导航 | `NavigationBar` + `NavigationBarItem` |
| 侧边导航 | `NavigationRail` / `NavigationDrawer` |
| 顶部栏 | `TopAppBar` / `CenterAlignedTopAppBar` |
| 对话框 / 模态框 | `AlertDialog` / `BasicAlertDialog` |
| 进度指示器 | `CircularProgressIndicator` / `LinearProgressIndicator` |
| 分隔线 | `HorizontalDivider` |
| 圆角图片 | `Image` + `Modifier.clip()` |
| 下拉菜单 | `ExposedDropdownMenuBox` |
| 滑块 | `Slider` / `RangeSlider` |

---

## 2. Figma-to-Compose 属性映射表

### 布局容器

| Figma 概念 | Compose 等效项 |
|---|---|
| Frame（无 auto-layout） | `Box` |
| Auto-layout Vertical | `Column` |
| Auto-layout Horizontal | `Row` |
| Auto-layout Wrap | `FlowRow` / `FlowColumn` |
| Grid（固定列数） | `LazyVerticalGrid(columns = GridCells.Fixed(n))` |
| 绝对定位子元素 | `Box` + `Modifier.offset(x, y)` 或 `Modifier.align()` |
| 带变体的组件 | 带参数的 Composable 函数 |
| 组件实例 | 函数调用点 |
| Section / Group（组织性的） | 不需要 composable；扁平化到父组件中 |

### 尺寸模式

| Figma 尺寸 | Compose Modifier |
|---|---|
| 固定宽度/高度 | `Modifier.size(w.dp, h.dp)` 或 `.width(w.dp).height(h.dp)` |
| Hug contents | 默认行为（包裹内容）-- 不需要 modifier |
| Fill container（水平） | `Modifier.fillMaxWidth()` |
| Fill container（垂直） | `Modifier.fillMaxHeight()` |
| Fill container（两者） | `Modifier.fillMaxSize()` |
| Fill 带最小宽度 | `Modifier.fillMaxWidth().widthIn(min = minW.dp)` |
| Fill 带最大宽度 | `Modifier.fillMaxWidth().widthIn(max = maxW.dp)` |
| Fill 带最小/最大高度 | `Modifier.fillMaxHeight().heightIn(min = ..., max = ...)` |
| 宽高比约束 | `Modifier.aspectRatio(ratio)` |

### 间距模型

**关键原则：父组件拥有间距。**

| Figma 间距 | Compose 等效项 |
|---|---|
| Padding（四边） | 在容器上使用 `Modifier.padding(all.dp)` |
| Padding（每边） | `Modifier.padding(start = ..., top = ..., end = ..., bottom = ...)` |
| 子元素间隙（垂直 auto-layout） | `Column(verticalArrangement = Arrangement.spacedBy(gap.dp))` |
| 子元素间隙（水平 auto-layout） | `Row(horizontalArrangement = Arrangement.spacedBy(gap.dp))` |
| Space between（分布） | `Arrangement.SpaceBetween` |
| Space around | `Arrangement.SpaceAround` |

**应做：** 为 RTL 语言支持使用 `start`/`end` 而非 `left`/`right`。

```kotlin
// 正确：RTL 感知
Modifier.padding(start = 16.dp, end = 8.dp)

// 错误：在 RTL 区域会失效
Modifier.padding(left = 16.dp, right = 8.dp)  // 避免
```

**应做：** 为统一间隙使用 `Arrangement.spacedBy()`。避免在每个子元素之间插入 `Spacer`。

```kotlin
// 应做：干净且统一
Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
    Text("First")
    Text("Second")
    Text("Third")
}

// 不应做：到处都是手动 spacer
Column {
    Text("First")
    Spacer(Modifier.height(12.dp))
    Text("Second")
    Spacer(Modifier.height(12.dp))
    Text("Third")
}
```

### 阴影映射（Compose 1.9+）

Compose Foundation 1.9 引入了 `dropShadow()` 和 `innerShadow()` 作为 modifier 扩展，替代了旧的 `shadow()` modifier 以实现精细控制。

```kotlin
// Drop shadow：放在 modifier 链中 background 之前
Box(
    Modifier
        .dropShadow(
            shape = RoundedCornerShape(12.dp),
            color = Color.Black.copy(alpha = 0.15f),
            blur = 8.dp,
            offsetX = 0.dp,
            offsetY = 4.dp,
            spread = 0.dp
        )
        .background(Color.White, RoundedCornerShape(12.dp))
        .padding(16.dp)
)

// Inner shadow：放在 modifier 链中 background 之后
Box(
    Modifier
        .background(Color.White, RoundedCornerShape(12.dp))
        .innerShadow(
            shape = RoundedCornerShape(12.dp),
            color = Color.Black.copy(alpha = 0.1f),
            blur = 4.dp,
            offsetX = 0.dp,
            offsetY = 2.dp,
            spread = 0.dp
        )
        .padding(16.dp)
)
```

**Figma 阴影字段到 Compose 映射：**

| Figma 阴影属性 | Compose 参数 |
|---|---|
| X offset | `offsetX` |
| Y offset | `offsetY` |
| Blur | `blur` |
| Spread | `spread` |
| Color + opacity | `color = Color(hex).copy(alpha = opacity)` |
| Drop Shadow 类型 | `Modifier.dropShadow()` |
| Inner Shadow 类型 | `Modifier.innerShadow()` |

**旧方法**（1.9 之前，对于简单 elevation 阴影仍然有效）：

```kotlin
// 简单 elevation 阴影
Box(
    Modifier
        .shadow(elevation = 4.dp, shape = RoundedCornerShape(12.dp))
        .background(Color.White)
)
```

### 渐变映射

| Figma 渐变类型 | Compose Brush |
|---|---|
| Linear gradient | `Brush.linearGradient(colors, start, end)` |
| Radial gradient | `Brush.radialGradient(colors, center, radius)` |
| Angular/sweep gradient | `Brush.sweepGradient(colors, center)` |

Figma 使用归一化坐标（0.0 到 1.0）。转换为像素 `Offset` 值：

```kotlin
// Figma linear gradient：从 start (0, 0) 到 end (1, 1)，45 度对角线
Box(
    Modifier
        .fillMaxWidth()
        .height(200.dp)
        .background(
            Brush.linearGradient(
                colors = listOf(Color(0xFF6200EE), Color(0xFF03DAC6)),
                start = Offset.Zero,
                end = Offset.Infinite  // diagonal
            )
        )
)

// 对于精确的 Figma 坐标，使用 onSizeChanged 或 BoxWithConstraints：
BoxWithConstraints(
    Modifier.background(
        Brush.linearGradient(
            colors = listOf(Color(0xFF6200EE), Color(0xFF03DAC6)),
            start = Offset(0f, 0f),
            end = Offset(constraints.maxWidth.toFloat(), constraints.maxHeight.toFloat())
        )
    )
) {
    // content
}

// Radial gradient
Box(
    Modifier
        .size(200.dp)
        .background(
            Brush.radialGradient(
                colors = listOf(Color.White, Color.Blue),
                center = Offset(100f, 100f),  // 200dp box 的中心（近似）
                radius = 150f
            )
        )
)
```

### Corner Radius

| Figma Corner Radius | Compose Shape |
|---|---|
| 所有圆角相等 | `RoundedCornerShape(radius.dp)` |
| 每圆角不同值 | `RoundedCornerShape(topStart = ..., topEnd = ..., bottomEnd = ..., bottomStart = ...)` |
| 完全圆角（胶囊） | `RoundedCornerShape(50)` 或 `CircleShape` |
| 无圆角 | `RectangleShape` |
| 切角 | `CutCornerShape(size.dp)` |

### Borders

```kotlin
// 实线边框
Modifier.border(width.dp, Color(0xFFCCCCCC), RoundedCornerShape(8.dp))

// 渐变边框
Modifier.border(
    width = 2.dp,
    brush = Brush.linearGradient(listOf(Color.Red, Color.Blue)),
    shape = RoundedCornerShape(8.dp)
)
```

### Opacity

| Figma 属性 | Compose 等效项 |
|---|---|
| Layer opacity | `Modifier.alpha(0.5f)` |
| Fill color opacity | `Color(0xFF000000).copy(alpha = 0.5f)` |
| Blend mode | `Modifier.graphicsLayer { compositingStrategy = ... }` |

### Image Fill Modes

| Figma Image Mode | Compose ContentScale |
|---|---|
| Fill（cover，可能裁剪） | `ContentScale.Crop` |
| Fit（contain，不裁剪） | `ContentScale.Fit` |
| Stretch（扭曲） | `ContentScale.FillBounds` |
| Tile | 自定义 `DrawScope` 平铺 |
| Fill width | `ContentScale.FillWidth` |
| Fill height | `ContentScale.FillHeight` |

```kotlin
Image(
    painter = painterResource(R.drawable.hero),
    contentDescription = "Hero image",
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .fillMaxWidth()
        .height(200.dp)
        .clip(RoundedCornerShape(12.dp))
)
```

---

## 3. Design Token 到 MaterialTheme 映射

| 设计系统 Token | MaterialTheme API |
|---|---|
| Color styles（Primary、Surface、Error...） | `MaterialTheme.colorScheme` |
| Text styles（Heading、Body、Caption...） | `MaterialTheme.typography` |
| Corner radius（Small、Medium、Large...） | `MaterialTheme.shapes` |
| Spacing scale（4、8、16、24...） | 自定义 `CompositionLocal`（见下方） |
| Elevation scale | 自定义 `CompositionLocal`（见下方） |

### 自定义间距 CompositionLocal

Material 3 不提供间距尺度。定义一个反映你的设计系统的：

```kotlin
@Immutable
data class AppSpacing(
    val xxs: Dp = 2.dp,
    val xs: Dp = 4.dp,
    val sm: Dp = 8.dp,
    val md: Dp = 16.dp,
    val lg: Dp = 24.dp,
    val xl: Dp = 32.dp,
    val xxl: Dp = 48.dp
)

val LocalAppSpacing = staticCompositionLocalOf { AppSpacing() }

// 在你的主题包装器中提供
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    val spacing = AppSpacing()
    CompositionLocalProvider(LocalAppSpacing provides spacing) {
        MaterialTheme(
            colorScheme = lightColorScheme(),
            typography = Typography(),
            shapes = Shapes()
        ) {
            content()
        }
    }
}

// 在调用点使用
@Composable
fun ProfileCard() {
    val spacing = LocalAppSpacing.current
    Card(
        modifier = Modifier.padding(spacing.md)
    ) {
        Column(
            modifier = Modifier.padding(spacing.md),
            verticalArrangement = Arrangement.spacedBy(spacing.sm)
        ) {
            Text("Name", style = MaterialTheme.typography.titleMedium)
            Text("Bio", style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

### 自定义 Elevation CompositionLocal

```kotlin
@Immutable
data class AppElevation(
    val none: Dp = 0.dp,
    val xs: Dp = 1.dp,
    val sm: Dp = 2.dp,
    val md: Dp = 4.dp,
    val lg: Dp = 8.dp,
    val xl: Dp = 16.dp
)

val LocalAppElevation = staticCompositionLocalOf { AppElevation() }
```

### 将 Figma Text Styles 映射到 Typography

```kotlin
// Figma design system:        Compose Typography:
// Heading/H1  36sp Bold    → displaySmall 或 headlineLarge
// Heading/H2  28sp Bold    → headlineMedium
// Heading/H3  22sp SemiBold→ titleLarge
// Body/Large  16sp Regular → bodyLarge
// Body/Small  14sp Regular → bodyMedium
// Caption     12sp Regular → bodySmall 或 labelMedium
// Button      14sp Medium  → labelLarge

val AppTypography = Typography(
    headlineLarge = TextStyle(
        fontFamily = yourFontFamily,
        fontWeight = FontWeight.Bold,
        fontSize = 36.sp,
        lineHeight = 44.sp
    ),
    // ... 映射每个 Figma text style
)
```

---

## 4. Modifier 排序规则

**规范原则：** 从外到内 = 先布局/尺寸，再装饰，再交互。

Modifier 在链中从左到右应用。每个 modifier 包裹其后的一切。将其想象为从外到内的层。

### 正确排序模式

```kotlin
// Pattern 1: 卡片式表面
Modifier
    .fillMaxWidth()                              // 1. 布局尺寸
    .padding(horizontal = 16.dp, vertical = 8.dp) // 2. 外边距（与兄弟元素的空间）
    .dropShadow(                                  // 3. 阴影（在 background 之前）
        shape = RoundedCornerShape(12.dp),
        color = Color.Black.copy(alpha = 0.1f),
        blur = 8.dp, offsetY = 4.dp
    )
    .background(Color.White, RoundedCornerShape(12.dp)) // 4. 背景填充
    .clip(RoundedCornerShape(12.dp))              // 5. 将内容裁剪为形状
    .clickable { }                                // 6. 交互（在 clip 内部，涟漪限制在边界内）
    .padding(16.dp)                               // 7. 内边距（内容内缩）

// Pattern 2: 带大触摸目标的 Clickable
Modifier
    .fillMaxWidth()
    .clickable { }           // Clickable 在 padding 之后 = 更大的触摸目标
    .padding(16.dp)          // 内部内容内边距

// Pattern 3: Background 延伸到边缘，padding 在内部
Modifier
    .fillMaxWidth()
    .background(MaterialTheme.colorScheme.surface)
    .padding(16.dp)
```

### 常见错误

```kotlin
// 错误：padding 在 fillMaxWidth 之前会裁剪填充区域
Modifier
    .padding(16.dp)
    .fillMaxWidth()  // 在 padding 应用后填充剩余宽度

// 正确：先 fillMaxWidth，再向内 pad
Modifier
    .fillMaxWidth()
    .padding(16.dp)

// 错误：clickable 在 padding 之前 = 触摸目标小
Modifier
    .clickable { }
    .padding(16.dp)   // Padding 在 clickable 区域之外

// 正确：clickable 在 padding 之后 = padding 区域也可点击
Modifier
    .padding(16.dp)
    .clickable { }    // 整个带 pad 的区域响应点击

// 错误：shadow 在 background 之后（不可见或被裁剪）
Modifier
    .background(Color.White, RoundedCornerShape(12.dp))
    .dropShadow(...)  // Shadow 绘制在 background 层内部

// 正确：shadow 在 background 之前
Modifier
    .dropShadow(...)
    .background(Color.White, RoundedCornerShape(12.dp))
```

---

## 5. 语义翻译 vs 字面翻译

Figma 设计表达视觉输出。Compose 代码应表达组件意图。始终优先使用 Material 3 组件而非用 `Box` + modifier 手动重建外观。

### 反模式：字面翻译

```kotlin
// Figma 展示一个卡片：圆角矩形、阴影、图片、标题、副标题
// 字面翻译 -- 不要这样做
Box(
    Modifier
        .shadow(4.dp, RoundedCornerShape(12.dp))
        .background(Color.White, RoundedCornerShape(12.dp))
        .clip(RoundedCornerShape(12.dp))
) {
    Column(Modifier.padding(16.dp)) {
        Image(painter = painterResource(R.drawable.photo), contentDescription = null)
        Text("Title", fontSize = 18.sp, fontWeight = FontWeight.Bold, color = Color.Black)
        Text("Subtitle", fontSize = 14.sp, color = Color(0xFF666666))
    }
}
```

### 正确：语义翻译

```kotlin
// 语义翻译 -- 应这样做
ElevatedCard(
    modifier = Modifier.fillMaxWidth(),
    shape = MaterialTheme.shapes.medium
) {
    Column {
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = "Photo description",
            contentScale = ContentScale.Crop,
            modifier = Modifier
                .fillMaxWidth()
                .height(180.dp)
        )
        Column(Modifier.padding(16.dp)) {
            Text("Title", style = MaterialTheme.typography.titleMedium)
            Text(
                "Subtitle",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

### 为什么这很重要

| 关注点 | 字面 Box+Modifiers | M3 组件 |
|---|---|---|
| 暗色主题 | 失效（硬编码颜色） | 自动适应 |
| Elevation overlay | 缺失 | 内置 tonal elevation |
| Ripple / 反馈 | 必须手动添加 | 内置 |
| Dynamic color（Material You） | 不响应 | 自动适应 |
| 状态处理（disabled、focused） | 手动处理 | 每状态内置样式 |

**规则：** 如果 Material 3 组件存在对应的视觉模式，使用它。只为真正新颖的 UI 元素构建自定义布局。

### 快速映射：视觉模式到 M3 组件

| "它看起来像一个..." | 使用 |
|---|---|
| 带阴影的圆角卡片 | `ElevatedCard` |
| 带边框的卡片 | `OutlinedCard` |
| 胶囊形按钮 | `Button(shape = CircleShape)` |
| 图标 + 标签行 | `ListItem` |
| 搜索栏 | `SearchBar` / `DockedSearchBar` |
| 分段控件 | `SegmentedButton`（M3 1.2+） |
| 横幅通知 | `Snackbar` |
| 全宽分隔线 | `HorizontalDivider` |
| 下拉刷新 | `PullToRefreshBox` |

---

## 6. 反模式

### 过度嵌套布局

Figma 设计通常为了组织目的有很深的画框层级。不要在 Compose 中镜像这种嵌套 -- 积极扁平化。

```kotlin
// 反模式：镜像 Figma 的 5 层嵌套
Box {
    Column {
        Row {
            Box {
                Column {
                    Text("Title")
                    Text("Subtitle")
                }
            }
        }
    }
}

// 正确：扁平化为布局实际需要的结构
Column {
    Text("Title", style = MaterialTheme.typography.titleMedium)
    Text("Subtitle", style = MaterialTheme.typography.bodyMedium)
}
```

深层嵌套增加 measure/layout 遍历次数。每个布局节点都是测量成本。扁平化为实现视觉结果所需的最小树深度。

### 硬编码值 vs 主题 Token

```kotlin
// 反模式：硬编码十六进制颜色和字体大小
Text(
    text = "Hello",
    color = Color(0xFF1A1A1A),
    fontSize = 16.sp,
    fontWeight = FontWeight.Medium
)

Box(Modifier.background(Color(0xFFF5F5F5)))

// 正确：主题 token
Text(
    text = "Hello",
    style = MaterialTheme.typography.bodyLarge,
    color = MaterialTheme.colorScheme.onSurface
)

Box(Modifier.background(MaterialTheme.colorScheme.surfaceVariant))
```

硬编码值会破坏暗色主题、动态颜色和设计系统更新。唯一可接受的硬编码颜色在你的主题定义文件中。

### 只为单一屏幕宽度设计

```kotlin
// 反模式：固定宽度在平板上会失效
Row(Modifier.width(360.dp)) {
    Column(Modifier.width(180.dp)) { /* left panel */ }
    Column(Modifier.width(180.dp)) { /* right panel */ }
}

// 正确：使用 WindowSizeClass 实现响应式
val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass

when (windowSizeClass.windowWidthSizeClass) {
    WindowWidthSizeClass.COMPACT -> {
        // 单列布局（手机）
        Column { /* 所有内容堆叠 */ }
    }
    WindowWidthSizeClass.MEDIUM -> {
        // 双面板列表-详情（小平板、折叠屏）
        ListDetailPaneScaffold(/* ... */)
    }
    WindowWidthSizeClass.EXPANDED -> {
        // 导航栏 + 内容（大平板、桌面）
        Row {
            NavigationRail { /* ... */ }
            Content(Modifier.weight(1f))
        }
    }
}
```

**应做：** 为响应式布局使用 `fillMaxWidth()`、`weight()` 和 `WindowSizeClass`。

**不应做：** 对应该自适应的容器使用固定像素/dp 宽度。

### 忘记滚动行为

```kotlin
// 反模式：内容溢出但无法滚动
Column(Modifier.fillMaxSize()) {
    // 20 个条目超出屏幕高度 -- 底部条目不可见
    repeat(20) { Text("Item $it") }
}

// 正确：添加滚动
Column(
    Modifier
        .fillMaxSize()
        .verticalScroll(rememberScrollState())
) {
    repeat(20) { Text("Item $it") }
}

// 或者对于动态列表：
LazyColumn(Modifier.fillMaxSize()) {
    items(20) { Text("Item $it") }
}
```

### 忽略 Scaffold 的 Content Padding

```kotlin
// 反模式：忽略 Scaffold 的 innerPadding
Scaffold(topBar = { TopAppBar(title = { Text("App") }) }) { innerPadding ->
    // 内容渲染在顶部栏后方
    LazyColumn {
        items(data) { Text(it) }
    }
}

// 正确：应用 innerPadding
Scaffold(topBar = { TopAppBar(title = { Text("App") }) }) { innerPadding ->
    LazyColumn(
        modifier = Modifier.padding(innerPadding),  // 或 contentPadding = innerPadding
    ) {
        items(data) { Text(it) }
    }
}
```
