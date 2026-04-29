# Jetpack Compose 动画

参考：`androidx/compose/animation/animation/src/commonMain/kotlin/androidx/compose/animation/`

## 基于状态的动画

### animate*AsState

通过目标值驱动单个属性的动画。值变化时动画启动。

```kotlin
val size by animateDpAsState(
    targetValue = if (isExpanded) 200.dp else 100.dp,
    animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
    label = "size"
)

Box(modifier = Modifier.size(size))
```

常用变体：

```kotlin
animateColorAsState(targetValue = Color.Blue)
animateFloatAsState(targetValue = 1f)
animateIntAsState(targetValue = 100)
animateOffsetAsState(targetValue = Offset(10f, 20f))
```

每个变体自动处理协程和重组。使用 `label` 参数便于调试。

## AnimatedVisibility

控制出现/消失动画，支持进入和退出过渡。

```kotlin
var visible by remember { mutableStateOf(true) }

AnimatedVisibility(visible = visible) {
    Text("Hello!")
}

// 触发
Button(onClick = { visible = !visible }) { Text("Toggle") }
```

### 进入/退出过渡

```kotlin
AnimatedVisibility(
    visible = visible,
    enter = slideInHorizontally(initialOffsetX = { -it }) + fadeIn(),
    exit = slideOutHorizontally(targetOffsetX = { -it }) + fadeOut()
) {
    Text("Animated!")
}
```

内置过渡：
- `slideInVertically`, `slideOutVertically`
- `slideInHorizontally`, `slideOutHorizontally`
- `expandVertically`, `shrinkVertically`
- `expandHorizontally`, `shrinkHorizontally`
- `fadeIn`, `fadeOut`
- `scaleIn`, `scaleOut`
- 使用 `+` 组合：`slideInVertically() + fadeIn()`

### 高级用法：自定义动画规格

```kotlin
AnimatedVisibility(
    visible = visible,
    enter = slideInVertically(
        initialOffsetY = { fullHeight -> fullHeight },
        animationSpec = spring()
    ),
    exit = slideOutVertically(
        targetOffsetY = { fullHeight -> fullHeight },
        animationSpec = tween(durationMillis = 300)
    )
) {
    Box(Modifier.fillMaxWidth().height(100.dp).background(Color.Blue))
}
```

## AnimatedContent

以平滑过渡替换内容。

```kotlin
var count by remember { mutableStateOf(0) }

AnimatedContent(targetState = count) { target ->
    Text(text = "Count: $target")
}

Button(onClick = { count++ }) { Text("Increment") }
```

### 自定义 transitionSpec

```kotlin
AnimatedContent(
    targetState = count,
    transitionSpec = {
        slideInVertically(initialOffsetY = { it }) with slideOutVertically(targetOffsetY = { -it })
    }
) { target ->
    Text("$target")
}
```

使用 `with` 同时指定退出和进入。这会同时运行退出和进入动画。

### 编排过渡顺序

```kotlin
AnimatedContent(
    targetState = count,
    transitionSpec = {
        slideInVertically(initialOffsetY = { it }) with slideOutVertically(targetOffsetY = { -it }) using SizeTransform(clip = false)
    }
) { target ->
    Text(
        "Count: $target",
        modifier = Modifier.fillMaxWidth()
    )
}
```

`SizeTransform` 在内容变化时平滑动画容器尺寸。

## Crossfade

简单的内容切换，带淡入淡出效果。

```kotlin
var showFirst by remember { mutableStateOf(true) }

Crossfade(targetState = showFirst) { state ->
    if (state) {
        Text("First")
    } else {
        Text("Second")
    }
}
```

`AnimatedContent` 的轻量级替代方案，用于简单的可见性切换。

## updateTransition

使用单一状态协调多个动画值。

```kotlin
var expanded by remember { mutableStateOf(false) }
val transition = updateTransition(targetState = expanded)

val size by transition.animateDp { if (it) 200.dp else 100.dp }
val color by transition.animateColor { if (it) Color.Blue else Color.Red }

Box(
    modifier = Modifier
        .size(size)
        .background(color)
        .clickable { expanded = !expanded }
)
```

所有动画同步运行，由单一状态变化控制。适用于具有多个动画属性的复杂组件。

## rememberInfiniteTransition

创建循环动画。

```kotlin
val infiniteTransition = rememberInfiniteTransition(label = "infinite")

val alpha by infiniteTransition.animateFloat(
    initialValue = 0f,
    targetValue = 1f,
    animationSpec = infiniteRepeatable(
        animation = tween(1000),
        repeatMode = RepeatMode.Reverse
    ),
    label = "alpha"
)

Text("Pulsing", modifier = Modifier.alpha(alpha))
```

持续运行直到 composable 被移除。非常适合加载状态、脉冲指示器。

## Animatable

在协程中进行命令式动画控制。用于精细控制。

```kotlin
val animatable = remember { Animatable(0f) }

LaunchedEffect(trigger) {
    animatable.animateTo(
        targetValue = 100f,
        animationSpec = spring()
    )
}

Box(Modifier.graphicsLayer(translationX = animatable.value))
```

适用于响应手势或复杂条件：

```kotlin
val animatable = remember { Animatable(0f) }

LaunchedEffect(Unit) {
    animatable.animateTo(targetValue = 360f, animationSpec = tween(2000))
}

Box(
    Modifier
        .size(100.dp)
        .background(Color.Blue)
        .graphicsLayer(rotationZ = animatable.value)
)
```

## 动画规格

### spring — 基于物理的真实感

```kotlin
val size by animateDpAsState(
    targetValue = 200.dp,
    animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy, stiffness = Spring.StiffnessLow)
)
```

- `dampingRatio`：`NoBouncy` (1f), `LowBouncy` (0.75f), `MediumBouncy` (0.5f), `HighBouncy` (0.2f)
- `stiffness`：`Low`, `Medium`, `High`

用于交互反馈，对用户来说感觉自然。

### tween — 基于时间

```kotlin
val color by animateColorAsState(
    targetValue = Color.Blue,
    animationSpec = tween(durationMillis = 500, easing = EaseInOutCubic)
)
```

缓动函数：`EaseInQuad`, `EaseOutQuad`, `EaseInOutQuad`, `LinearEasing`, `FastOutSlowInEasing`。

可预测的时间，适合顺序动画。

### keyframes — 逐帧控制

```kotlin
val position by animateFloatAsState(
    targetValue = 100f,
    animationSpec = keyframes {
        0f at 0 using EaseInQuad
        50f at 150 using EaseOutQuad
        100f at 300
    }
)
```

在特定时间戳定义精确值。用于复杂编排。

## 自动尺寸动画

### animateContentSize

内容变化时平滑动画 Box 尺寸。

```kotlin
var expanded by remember { mutableStateOf(false) }

Box(
    modifier = Modifier
        .animateContentSize()
        .background(Color.Blue)
        .clickable { expanded = !expanded }
) {
    Column {
        Text("Header")
        if (expanded) {
            Text("Expanded content...")
        }
    }
}
```

不需要显式的 `AnimatedVisibility` 或布局过渡。自动处理容器。

## LazyLists 中的布局动画

### animateItem — 替代 animateItemPlacement

动画化条目的出现、移除和重新排序。

```kotlin
LazyColumn {
    items(items, key = { it.id }) { item ->
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .animateItem()
                .padding(8.dp)
                .background(Color.Gray)
        ) {
            Text(item.name)
        }
    }
}
```

自动动画：
- 新条目滑入
- 移除条目滑出
- 重新排序条目移动到新位置

在 Lazy 布局（LazyColumn、LazyRow、LazyVerticalGrid）中的条目上调用。

## 共享元素过渡

使用 `SharedTransitionLayout` 和 Navigation Compose 跨屏幕边界无缝动画元素。

### sharedElement() vs sharedBounds()

| 方面 | `sharedElement()` | `sharedBounds()` |
|---|---|---|
| **内容** | 两个屏幕上相同（同一张图片、同一个图标） | 源和目标内容不同（例如卡片展开为详情页） |
| **用例** | 主图、头像、缩略图 | 容器变换、卡片到页面 |
| **过渡期间** | 只渲染目标 composable | 源和目标都可见并交叉淡入淡出 |

### 完整工作示例

```kotlin
@Composable
fun App() {
    SharedTransitionLayout {
        NavHost(navController = navController, startDestination = "list") {
            composable("list") {
                ListScreen(
                    onItemClick = { id -> navController.navigate("detail/$id") },
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable
                )
            }
            composable("detail/{id}") { backStackEntry ->
                val id = backStackEntry.arguments?.getString("id") ?: return@composable
                DetailScreen(
                    itemId = id,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable
                )
            }
        }
    }
}

@Composable
fun ListScreen(
    onItemClick: (String) -> Unit,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    with(sharedTransitionScope) {
        Row(
            modifier = Modifier
                .clickable { onItemClick(item.id) }
                // sharedBounds 包裹整个卡片容器（源/目标内容不同）
                .sharedBounds(
                    sharedContentState = rememberSharedContentState(key = "card-${item.id}"),
                    animatedVisibilityScope = animatedVisibilityScope,
                    boundsTransform = BoundsTransform { initialBounds, targetBounds ->
                        keyframes {
                            durationMillis = 500
                            initialBounds at 0 using ArcMode.ArcBelow
                            targetBounds at 500
                        }
                    }
                )
        ) {
            Image(
                painter = painterResource(item.imageRes),
                contentDescription = null,
                modifier = Modifier
                    .size(80.dp)
                    // sharedElement 用于跨屏幕相同的图片
                    .sharedElement(
                        state = rememberSharedContentState(key = "image-${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
            )
            Text(
                text = item.title,
                modifier = Modifier
                    .sharedElement(
                        state = rememberSharedContentState(key = "title-${item.id}"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                    // 通过立即对齐到最终尺寸防止文本重排
                    .skipToLookaheadSize()
            )
        }
    }
}

@Composable
fun DetailScreen(
    itemId: String,
    sharedTransitionScope: SharedTransitionScope,
    animatedVisibilityScope: AnimatedVisibilityScope
) {
    with(sharedTransitionScope) {
        Column(
            modifier = Modifier
                .sharedBounds(
                    sharedContentState = rememberSharedContentState(key = "card-$itemId"),
                    animatedVisibilityScope = animatedVisibilityScope
                )
        ) {
            Image(
                painter = painterResource(item.imageRes),
                contentDescription = null,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(300.dp)
                    .sharedElement(
                        state = rememberSharedContentState(key = "image-$itemId"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
            )
            Text(
                text = item.title,
                style = MaterialTheme.typography.headlineMedium,
                modifier = Modifier
                    .sharedElement(
                        state = rememberSharedContentState(key = "title-$itemId"),
                        animatedVisibilityScope = animatedVisibilityScope
                    )
                    .skipToLookaheadSize()
            )
            // 非共享内容淡入
            Text(
                text = item.description,
                modifier = Modifier.animateEnterExit(
                    enter = fadeIn() + slideInVertically { it / 3 },
                    exit = fadeOut()
                )
            )
        }
    }
}
```

### BoundsTransform 实现弧形运动

控制源和目标边界之间的动画路径：

```kotlin
val arcBoundsTransform = BoundsTransform { initialBounds, targetBounds ->
    keyframes {
        durationMillis = 500
        initialBounds at 0 using ArcMode.ArcBelow
        targetBounds at 500
    }
}

// 应用到 sharedElement 或 sharedBounds
Modifier.sharedElement(
    state = rememberSharedContentState(key = "hero"),
    animatedVisibilityScope = animatedVisibilityScope,
    boundsTransform = arcBoundsTransform
)
```

### 叠加层渲染

过渡期间将共享元素保持在所有其他内容之上：

```kotlin
Modifier.sharedElement(
    state = rememberSharedContentState(key = "fab"),
    animatedVisibilityScope = animatedVisibilityScope,
    renderInSharedTransitionScopeOverlay = true // 在导航过渡之上渲染
)
```

### 防止文本重排

使用 `skipToLookaheadSize()` 让文本 composable 立即对齐到最终尺寸，避免过渡中途出现尴尬的换行变化：

```kotlin
Text(
    text = item.title,
    modifier = Modifier
        .sharedElement(
            state = rememberSharedContentState(key = "title-${item.id}"),
            animatedVisibilityScope = animatedVisibilityScope
        )
        .skipToLookaheadSize() // 文本立即使用目标尺寸，无重排
)
```

## 性能：transform 使用 graphicsLayer

使用 `graphicsLayer` 动画变换而非布局变化。

```kotlin
// ✅ 正确：使用 GPU 加速的 graphicsLayer
val offset by animateFloatAsState(targetValue = 100f)
Box(modifier = Modifier.graphicsLayer(translationX = offset))

// ❌ 避免：导致重组和重新布局
val offset by animateFloatAsState(targetValue = 100f)
Box(modifier = Modifier.offset(x = offset.dp))
```

对以下情况使用 `graphicsLayer`：
- 平移（`translationX`, `translationY`）
- 旋转（`rotationX`, `rotationY`, `rotationZ`）
- 缩放（`scaleX`, `scaleY`）
- Alpha（不透明度）

## 反模式

### 不要：用 if 动画化可见性

```kotlin
// ❌ 反模式
@Composable
fun MyScreen() {
    if (visible) {
        Text("Content") // 无动画直接出现/消失
    }
}

// ✅ 正确
@Composable
fun MyScreen() {
    AnimatedVisibility(visible = visible) {
        Text("Content")
    }
}
```

### 不要：在 composition 中创建 Animatable

```kotlin
// ❌ 反模式
@Composable
fun MyScreen() {
    val animatable = Animatable(0f) // 每次重组都重新创建！

    LaunchedEffect(Unit) {
        animatable.animateTo(100f)
    }
}

// ✅ 正确
@Composable
fun MyScreen() {
    val animatable = remember { Animatable(0f) } // 跨重组保持

    LaunchedEffect(Unit) {
        animatable.animateTo(100f)
    }
}
```

### 不要：在 composition 阶段动画

```kotlin
// ❌ 反模式
@Composable
fun MyScreen() {
    var position by remember { mutableStateOf(0f) }
    position = position + 10f // 无限重组循环！
}

// ✅ 正确
@Composable
fun MyScreen() {
    var position by remember { mutableStateOf(0f) }

    LaunchedEffect(Unit) {
        repeat(10) {
            position += 10f
            delay(16)
        }
    }
}
```

### 不要：忘记 label 参数

```kotlin
// ❌ 反模式（更难调试）
val size by animateDpAsState(targetValue = 100.dp)

// ✅ 正确
val size by animateDpAsState(
    targetValue = 100.dp,
    label = "box_size"
)
```

Labels 有助于调试布局检查器和动画检查工具。

---

## 动画决策树

### 何时使用哪个 API

| API | 使用场景 |
|---|---|
| `animate*AsState` | 由状态驱动的单个属性（尺寸、颜色、alpha）动画 |
| `AnimatedVisibility` | 以进入/退出过渡显示或隐藏 composable |
| `AnimatedContent` / `Crossfade` | 在不同 composable 之间切换（内容交换） |
| `updateTransition` | 多个属性必须从同一状态同步动画 |
| `Animatable` | 手势驱动或命令式控制（基于协程，支持 `snapTo`、`animateDecay`） |
| `rememberInfiniteTransition` | 无限循环动画（脉冲、旋转、闪烁） |
| `animateContentSize` | 内容变化时平滑动画容器尺寸 |
| `animateItem` | Lazy 布局中条目的出现、消失和重新排序 |

### 每个动画影响的阶段

Compose 渲染有三个阶段：**Composition**（显示什么）、**Layout**（放置位置）、**Draw**（如何渲染）。动画应在尽可能晚的阶段读取状态以最小化工作量。

```kotlin
// 最佳：仅 Draw 阶段 — 无重新布局，无重组
val alpha by animateFloatAsState(targetValue = if (visible) 1f else 0f, label = "alpha")
Box(
    modifier = Modifier.graphicsLayer { this.alpha = alpha }
)

// 良好：仅 Layout 阶段 — 重新布局但无重组
val offsetPx by animateIntAsState(targetValue = if (moved) 300 else 0, label = "offset")
Box(
    modifier = Modifier.offset { IntOffset(offsetPx, 0) }
)

// 一般：Composition + Layout — 每帧触发重组
val offsetDp by animateDpAsState(targetValue = if (moved) 100.dp else 0.dp, label = "offset")
Box(
    modifier = Modifier.offset(x = offsetDp)
)
```

**规则：** 将状态读取推迟到尽可能晚的阶段。使用基于 lambda 的 modifier（`graphicsLayer { }`、`offset { }`）而非基于参数的 modifier（`graphicsLayer(alpha = ...)`、`offset(x = ...)`）。

---

## 设计到动画的翻译

### Figma 缓动曲线到 Compose

| Figma Easing | Compose 等效项 |
|---|---|
| Linear | `LinearEasing` |
| Ease In | `FastOutLinearInEasing` |
| Ease Out | `LinearOutSlowInEasing` |
| Ease In and Out | `FastOutSlowInEasing` |
| Custom Bezier (x1, y1, x2, y2) | `CubicBezierEasing(x1, y1, x2, y2)` |

### M3 Motion Duration Token

| Token | 时长 |
|---|---|
| Short1 | 50ms |
| Short2 | 100ms |
| Short3 | 150ms |
| Short4 | 200ms |
| Medium1 | 250ms |
| Medium2 | 300ms |
| Medium3 | 350ms |
| Medium4 | 400ms |
| Long1 | 450ms |
| Long2 | 500ms |
| Long3 | 550ms |
| Long4 | 600ms |
| ExtraLong1 | 700ms |
| ExtraLong2 | 800ms |
| ExtraLong3 | 900ms |
| ExtraLong4 | 1000ms |

### M3 Easing Token

| Token | Compose 值 |
|---|---|
| Emphasized | `CubicBezierEasing(0.2f, 0f, 0f, 1f)` |
| EmphasizedDecelerate | `CubicBezierEasing(0.05f, 0.7f, 0.1f, 1f)` |
| EmphasizedAccelerate | `CubicBezierEasing(0.3f, 0f, 0.8f, 0.15f)` |
| Standard | `FastOutSlowInEasing` |
| StandardDecelerate | `LinearOutSlowInEasing` |
| StandardAccelerate | `FastOutLinearInEasing` |

### Spring 参数直觉

**Stiffness**（动画向目标移动的速度）：

| 值 | 常量 | 感觉 |
|---|---|---|
| ~26f | — | 缓慢、沉重、慵懒 |
| 200f | `Spring.StiffnessLow` | 柔和、放松 |
| 400f | `Spring.StiffnessMediumLow` | 随意、舒适 |
| 1500f | `Spring.StiffnessMedium` | 响应迅速、默认 |
| 10000f | `Spring.StiffnessHigh` | 干脆、即时 |

**Damping Ratio**（弹跳程度）：

| 值 | 常量 | 感觉 |
|---|---|---|
| 1.0f | `Spring.DampingRatioNoBouncy` | 无超调，直接稳定 |
| 0.75f | `Spring.DampingRatioLowBouncy` | 微妙弹跳，专业感 |
| 0.5f | `Spring.DampingRatioMediumBouncy` | 活泼，明显弹跳 |
| 0.2f | `Spring.DampingRatioHighBouncy` | 夸张，卡通感弹跳 |

### Figma Spring 到 Compose 转换

```kotlin
fun figmaSpringToCompose(mass: Float, stiffness: Float, damping: Float): SpringSpec<Float> {
    val dampingRatio = damping / (2f * sqrt(stiffness * mass))
    return spring(dampingRatio = dampingRatio, stiffness = stiffness)
}
```

### 生产验证的 Spring 规格

```kotlin
val figmaMatchedSpring = spring<Float>(dampingRatio = 0.444f, stiffness = 26.5f)
val responsiveSpring = spring<Float>(dampingRatio = 0.7f, stiffness = 800f)
val snappySpring = spring<Float>(dampingRatio = 0.6f, stiffness = 1000f)
```

---

## 手势驱动动画

### 使用 Animatable 的滑动删除

```kotlin
fun Modifier.swipeToDismiss(onDismiss: () -> Unit): Modifier = composed {
    val offsetX = remember { Animatable(0f) }
    val decay = rememberSplineBasedDecay<Float>()

    pointerInput(Unit) {
        coroutineScope {
            while (true) {
                val velocityTracker = VelocityTracker()
                // 等待触摸按下
                val pointerId = awaitPointerEventScope {
                    awaitFirstDown().id
                }
                // 取消任何进行中的动画
                offsetX.stop()

                awaitPointerEventScope {
                    horizontalDrag(pointerId) { change ->
                        val horizontalDragOffset = offsetX.value + change.positionChange().x
                        launch { offsetX.snapTo(horizontalDragOffset) }
                        velocityTracker.addPosition(change.uptimeMillis, change.position)
                        change.consume()
                    }
                }

                val velocity = velocityTracker.calculateVelocity().x
                val targetOffsetX = decay.calculateTargetValue(offsetX.value, velocity)

                offsetX.updateBounds(
                    lowerBound = -size.width.toFloat(),
                    upperBound = size.width.toFloat()
                )

                launch {
                    if (abs(targetOffsetX) >= size.width * 0.5f) {
                        // 甩得足够远 — 删除
                        offsetX.animateDecay(velocity, decay)
                        onDismiss()
                    } else {
                        // 弹回
                        offsetX.animateTo(
                            targetValue = 0f,
                            initialVelocity = velocity
                        )
                    }
                }
            }
        }
    }.offset { IntOffset(offsetX.value.roundToInt(), 0) }
}
```

### AnchoredDraggable 吸附点

```kotlin
enum class DragValue { Start, Center, End }

@Composable
fun AnchoredDraggableExample() {
    val density = LocalDensity.current
    val anchors = with(density) {
        DraggableAnchors {
            DragValue.Start at -200.dp.toPx()
            DragValue.Center at 0f
            DragValue.End at 200.dp.toPx()
        }
    }

    val state = remember {
        AnchoredDraggableState(
            initialValue = DragValue.Center,
            anchors = anchors,
            positionalThreshold = { totalDistance -> totalDistance * 0.5f },
            velocityThreshold = { with(density) { 125.dp.toPx() } },
            animationSpec = spring()
        )
    }

    Box(
        modifier = Modifier
            .offset { IntOffset(state.requireOffset().roundToInt(), 0) }
            .anchoredDraggable(state, Orientation.Horizontal)
            .size(80.dp)
            .background(Color.Blue, RoundedCornerShape(16.dp))
    )
}
```

### Transformable：捏合、缩放、旋转

```kotlin
@Composable
fun TransformableExample() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val transformableState = rememberTransformableState { zoomChange, offsetChange, rotationChange ->
        scale = (scale * zoomChange).coerceIn(0.5f, 5f)
        rotation += rotationChange
        offset += offsetChange
    }

    Box(
        modifier = Modifier
            .graphicsLayer {
                scaleX = scale
                scaleY = scale
                rotationZ = rotation
                translationX = offset.x
                translationY = offset.y
            }
            .transformable(state = transformableState)
            .size(200.dp)
            .background(Color.Blue)
    )
}
```

---

## 动画配方

### 闪烁 / 骨架屏加载

```kotlin
fun Modifier.shimmerEffect(): Modifier = composed {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val translateAnim by transition.animateFloat(
        initialValue = -1000f,
        targetValue = 1000f,
        animationSpec = infiniteRepeatable(
            animation = tween(durationMillis = 1200, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "shimmer_translate"
    )

    val shimmerBrush = Brush.linearGradient(
        colors = listOf(
            Color.LightGray.copy(alpha = 0.6f),
            Color.LightGray.copy(alpha = 0.2f),
            Color.LightGray.copy(alpha = 0.6f)
        ),
        start = Offset(translateAnim, 0f),
        end = Offset(translateAnim + 500f, 0f)
    )

    background(shimmerBrush)
}

@Composable
fun SkeletonCard() {
    Column(modifier = Modifier.padding(16.dp)) {
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp)
                .clip(RoundedCornerShape(12.dp))
                .shimmerEffect()
        )
        Spacer(modifier = Modifier.height(8.dp))
        Box(
            modifier = Modifier
                .fillMaxWidth(0.7f)
                .height(20.dp)
                .clip(RoundedCornerShape(4.dp))
                .shimmerEffect()
        )
    }
}

@Composable
fun ContentWithLoading(isLoading: Boolean, content: @Composable () -> Unit) {
    Crossfade(targetState = isLoading, label = "loading_crossfade") { loading ->
        if (loading) {
            SkeletonCard()
        } else {
            content()
        }
    }
}
```

### 交错列表入场

```kotlin
@Composable
fun StaggeredListEntrance(items: List<String>) {
    Column {
        items.forEachIndexed { index, item ->
            val animatable = remember { Animatable(0f) }
            LaunchedEffect(Unit) {
                delay(index * 100L)
                animatable.animateTo(
                    targetValue = 1f,
                    animationSpec = spring(
                        dampingRatio = Spring.DampingRatioLowBouncy,
                        stiffness = Spring.StiffnessMediumLow
                    )
                )
            }
            Text(
                text = item,
                modifier = Modifier
                    .graphicsLayer {
                        alpha = animatable.value
                        translationX = (1f - animatable.value) * 100f
                    }
                    .padding(8.dp)
            )
        }
    }
}
```

### 滑动删除（Material 3）

```kotlin
@Composable
fun SwipeToDismissItem(
    onDismiss: () -> Unit,
    content: @Composable () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value != SwipeToDismissBoxValue.Settled) {
                onDismiss()
                true
            } else false
        }
    )

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            val color by animateColorAsState(
                targetValue = when (dismissState.targetValue) {
                    SwipeToDismissBoxValue.StartToEnd -> Color.Green
                    SwipeToDismissBoxValue.EndToStart -> Color.Red
                    SwipeToDismissBoxValue.Settled -> Color.Transparent
                },
                label = "dismiss_bg"
            )
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(color)
                    .padding(horizontal = 20.dp),
                contentAlignment = when (dismissState.targetValue) {
                    SwipeToDismissBoxValue.StartToEnd -> Alignment.CenterStart
                    else -> Alignment.CenterEnd
                }
            ) {
                Icon(
                    imageVector = when (dismissState.targetValue) {
                        SwipeToDismissBoxValue.StartToEnd -> Icons.Default.Done
                        else -> Icons.Default.Delete
                    },
                    contentDescription = null,
                    tint = Color.White
                )
            }
        }
    ) {
        content()
    }
}
```

### 可展开卡片

```kotlin
@Composable
fun ExpandableCard(title: String, description: String) {
    var expanded by remember { mutableStateOf(false) }
    val arrowRotation by animateFloatAsState(
        targetValue = if (expanded) 180f else 0f,
        label = "arrow_rotation"
    )

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .animateContentSize(animationSpec = spring(stiffness = Spring.StiffnessMediumLow))
            .clickable { expanded = !expanded }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(text = title, style = MaterialTheme.typography.titleMedium, modifier = Modifier.weight(1f))
                Icon(
                    imageVector = Icons.Default.KeyboardArrowDown,
                    contentDescription = if (expanded) "Collapse" else "Expand",
                    modifier = Modifier.graphicsLayer { rotationZ = arrowRotation }
                )
            }
            AnimatedVisibility(visible = expanded) {
                Text(
                    text = description,
                    style = MaterialTheme.typography.bodyMedium,
                    modifier = Modifier.padding(top = 8.dp)
                )
            }
        }
    }
}
```

### 自定义下拉刷新

```kotlin
@Composable
fun CustomPullToRefresh(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    content: @Composable () -> Unit
) {
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh,
        indicator = { state ->
            val distanceFraction = state.distanceFraction.coerceIn(0f, 1f)
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(top = 16.dp),
                contentAlignment = Alignment.TopCenter
            ) {
                Icon(
                    imageVector = Icons.Default.Refresh,
                    contentDescription = "Refreshing",
                    modifier = Modifier
                        .size(32.dp)
                        .graphicsLayer {
                            scaleX = distanceFraction
                            scaleY = distanceFraction
                            rotationZ = distanceFraction * 360f
                        }
                )
            }
        }
    ) {
        content()
    }
}
```

### FAB 变形

**模式 1：ExtendedFloatingActionButton 配合滚动驱动的展开/收起**

```kotlin
@Composable
fun CollapsibleFab(listState: LazyListState) {
    val expandedFab by remember {
        derivedStateOf { listState.firstVisibleItemIndex == 0 }
    }

    ExtendedFloatingActionButton(
        onClick = { /* action */ },
        expanded = expandedFab,
        icon = { Icon(Icons.Default.Add, contentDescription = "Add") },
        text = { Text("New Item") }
    )
}
```

**模式 2：使用 updateTransition 的爆炸式 FAB**

```kotlin
@Composable
fun ExplodingFab(isExpanded: Boolean, onClick: () -> Unit) {
    val transition = updateTransition(targetState = isExpanded, label = "fab_explode")

    val size by transition.animateDp(label = "size") { if (it) 200.dp else 56.dp }
    val cornerRadius by transition.animateDp(label = "corner") { if (it) 16.dp else 28.dp }
    val color by transition.animateColor(label = "color") {
        if (it) MaterialTheme.colorScheme.secondaryContainer
        else MaterialTheme.colorScheme.primaryContainer
    }
    val contentAlpha by transition.animateFloat(label = "alpha") { if (it) 1f else 0f }

    Surface(
        modifier = Modifier.size(size).clickable { onClick() },
        shape = RoundedCornerShape(cornerRadius),
        color = color
    ) {
        Box(contentAlignment = Alignment.Center) {
            if (!isExpanded) {
                Icon(Icons.Default.Add, contentDescription = "Add")
            }
            Column(
                modifier = Modifier.graphicsLayer { alpha = contentAlpha },
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                // 展开内容
                Text("Option 1")
                Text("Option 2")
                Text("Option 3")
            }
        }
    }
}
```

### 底部浮层拖拽

```kotlin
enum class SheetValue { Hidden, Collapsed, Expanded }

@Composable
fun DraggableBottomSheet(content: @Composable () -> Unit) {
    val density = LocalDensity.current
    val anchors = with(density) {
        DraggableAnchors {
            SheetValue.Hidden at 0f
            SheetValue.Collapsed at -200.dp.toPx()
            SheetValue.Expanded at -600.dp.toPx()
        }
    }

    val state = remember {
        AnchoredDraggableState(
            initialValue = SheetValue.Hidden,
            anchors = anchors,
            positionalThreshold = { totalDistance -> totalDistance * 0.5f },
            velocityThreshold = { with(density) { 125.dp.toPx() } },
            animationSpec = spring(stiffness = Spring.StiffnessMediumLow)
        )
    }

    Box(modifier = Modifier.fillMaxSize()) {
        content()

        Surface(
            modifier = Modifier
                .fillMaxWidth()
                .align(Alignment.BottomCenter)
                .offset { IntOffset(0, (state.requireOffset()).roundToInt()) }
                .anchoredDraggable(state, Orientation.Vertical),
            shape = RoundedCornerShape(topStart = 16.dp, topEnd = 16.dp),
            shadowElevation = 8.dp
        ) {
            Column(modifier = Modifier.fillMaxWidth().height(600.dp).padding(16.dp)) {
                // 拖拽把手
                Box(
                    modifier = Modifier
                        .align(Alignment.CenterHorizontally)
                        .width(40.dp)
                        .height(4.dp)
                        .background(Color.Gray, RoundedCornerShape(2.dp))
                )
                Spacer(modifier = Modifier.height(16.dp))
                Text("Sheet Content")
            }
        }
    }
}
```

### 视差滚动头部

```kotlin
@Composable
fun ParallaxHeader(scrollState: ScrollState) {
    val scrollOffset = scrollState.value.toFloat()

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(300.dp)
            .graphicsLayer {
                translationY = scrollOffset * 0.6f // 视差因子
                scaleX = 1f + (scrollOffset * 0.001f).coerceAtLeast(0f)
                scaleY = 1f + (scrollOffset * 0.001f).coerceAtLeast(0f)
                alpha = (1f - (scrollOffset / 600f)).coerceIn(0f, 1f)
            }
    ) {
        Image(
            painter = painterResource(R.drawable.header),
            contentDescription = null,
            contentScale = ContentScale.Crop,
            modifier = Modifier.fillMaxSize()
        )
    }
}
```

### 动画标签切换

```kotlin
@Composable
fun AnimatedTabContent(selectedTabIndex: Int) {
    AnimatedContent(
        targetState = selectedTabIndex,
        transitionSpec = {
            val direction = if (targetState > initialState) 1 else -1
            slideInHorizontally(
                initialOffsetX = { fullWidth -> direction * fullWidth },
                animationSpec = tween(300)
            ) + fadeIn(animationSpec = tween(300)) togetherWith
            slideOutHorizontally(
                targetOffsetX = { fullWidth -> -direction * fullWidth },
                animationSpec = tween(300)
            ) + fadeOut(animationSpec = tween(300)) using
            SizeTransform(clip = false)
        },
        label = "tab_content"
    ) { tabIndex ->
        when (tabIndex) {
            0 -> TabOneContent()
            1 -> TabTwoContent()
            2 -> TabThreeContent()
        }
    }
}
```

---

## 顺序/并行动画编排

### 顺序（协程链式调用）

每个 `animateTo` 挂起直到完成，因此链式调用创建顺序动画：

```kotlin
val alpha = remember { Animatable(0f) }
val translateY = remember { Animatable(100f) }
val scale = remember { Animatable(0.5f) }

LaunchedEffect(Unit) {
    alpha.animateTo(1f, animationSpec = tween(300))
    translateY.animateTo(0f, animationSpec = spring())
    scale.animateTo(1f, animationSpec = tween(200))
}
```

### 并行（多个 launch 块）

```kotlin
val alpha = remember { Animatable(0f) }
val translateY = remember { Animatable(100f) }

LaunchedEffect(Unit) {
    coroutineScope {
        launch { alpha.animateTo(1f, animationSpec = tween(300)) }
        launch { translateY.animateTo(0f, animationSpec = spring()) }
    }
    // 这里的代码在两个动画都完成后运行
}
```

### 交错延迟

```kotlin
val items = remember { List(5) { Animatable(0f) } }

LaunchedEffect(Unit) {
    items.forEachIndexed { index, animatable ->
        launch {
            delay(index * 80L)
            animatable.animateTo(1f, animationSpec = spring())
        }
    }
}
```

### 混合顺序 + 并行

```kotlin
LaunchedEffect(Unit) {
    // 阶段 1：顺序 — 先淡入
    alpha.animateTo(1f, animationSpec = tween(200))

    // 阶段 2：并行 — 同时移动和缩放
    coroutineScope {
        launch { translateY.animateTo(0f, animationSpec = spring()) }
        launch { scale.animateTo(1f, animationSpec = spring()) }
    }

    // 阶段 3：顺序 — 阶段 2 完成后的最终 flourish
    rotation.animateTo(360f, animationSpec = tween(400))
}
```

---

## 预测性返回手势动画（Android 14+）

### NavHost 过渡

```kotlin
NavHost(
    navController = navController,
    startDestination = "home",
    enterTransition = {
        slideInHorizontally(initialOffsetX = { it }) + fadeIn(animationSpec = tween(300))
    },
    exitTransition = {
        slideOutHorizontally(targetOffsetX = { -it / 3 }) + fadeOut(animationSpec = tween(300))
    },
    popEnterTransition = {
        slideInHorizontally(initialOffsetX = { -it / 3 }) + fadeIn(animationSpec = tween(300))
    },
    popExitTransition = {
        slideOutHorizontally(targetOffsetX = { it }) + fadeOut(animationSpec = tween(300))
    }
) {
    composable("home") { HomeScreen() }
    composable("detail") { DetailScreen() }
}
```

### PredictiveBackHandler

```kotlin
@Composable
fun PredictiveBackExample(onBack: () -> Unit) {
    var boxScale by remember { mutableFloatStateOf(1f) }

    PredictiveBackHandler(enabled = true) { progress: Flow<BackEventCompat> ->
        try {
            progress.collect { backEvent ->
                boxScale = 1f - (0.3f * backEvent.progress)
            }
            onBack()
        } catch (e: CancellationException) {
            boxScale = 1f
            throw e
        }
    }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .graphicsLayer {
                scaleX = boxScale
                scaleY = boxScale
            }
    ) {
        Text("Swipe back to see scale animation")
    }
}
```

### M3 自动预测性返回

这些 Material 3 组件开箱即用地支持预测性返回手势动画（无需额外代码）：

- `SearchBar` — 滑动时折叠返回
- `ModalBottomSheet` — 随手势进度向下滑动
- `ModalNavigationDrawer` — 随手势进度滑动关闭

---

## 其他反模式

### 不要：在 draw 阶段足够时在 composition 中读取动画状态

```kotlin
// BAD：在 composition 期间读取 alpha，每帧触发重组
val alpha by animateFloatAsState(targetValue = 0.5f, label = "alpha")
Box(modifier = Modifier.alpha(alpha))

// GOOD：仅在 draw 阶段读取 alpha，跳过重组
val alpha by animateFloatAsState(targetValue = 0.5f, label = "alpha")
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha })
```

### 不要：使用 offset(x, y) 进行动画移动

```kotlin
// BAD：基于参数的 offset 触发重组 + 重新布局
val animatedDp by animateDpAsState(targetValue = 100.dp, label = "x")
Box(modifier = Modifier.offset(x = animatedDp))

// BETTER：基于 lambda 的 offset — 仅 layout 阶段，无重组
val animatedPx by animateIntAsState(targetValue = 300, label = "x")
Box(modifier = Modifier.offset { IntOffset(animatedPx, 0) })

// BEST：graphicsLayer — 仅 draw 阶段
val animatedPx by animateFloatAsState(targetValue = 300f, label = "x")
Box(modifier = Modifier.graphicsLayer { translationX = animatedPx })
```

### 不要：对独立属性使用 updateTransition

```kotlin
// BAD：属性不需要同步但被耦合
val transition = updateTransition(targetState = state, label = "t")
val alpha by transition.animateFloat(label = "a") { if (it) 1f else 0f }
val size by transition.animateDp(label = "s") { if (it) 200.dp else 100.dp }

// GOOD：独立属性使用独立的 animate*AsState
val alpha by animateFloatAsState(targetValue = if (state) 1f else 0f, label = "alpha")
val size by animateDpAsState(targetValue = if (state) 200.dp else 100.dp, label = "size")
```

### 不要：硬编码任意时长

```kotlin
// BAD：无设计依据的任意时长
val anim by animateFloatAsState(
    targetValue = 1f,
    animationSpec = tween(durationMillis = 347),
    label = "anim"
)

// GOOD：使用 M3 motion token 保持一致性
val anim by animateFloatAsState(
    targetValue = 1f,
    animationSpec = tween(durationMillis = MotionTokens.DurationMedium2.toInt()),
    label = "anim"
)

// BETTER：使用 spring() 实现可中断、自然感的动画
val anim by animateFloatAsState(
    targetValue = 1f,
    animationSpec = spring(stiffness = Spring.StiffnessMedium),
    label = "anim"
)
```
