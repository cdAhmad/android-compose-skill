# Jetpack Compose Modifier 参考

Modifier 是装饰或增强 composable 的主要方式。它们应用布局、绘制、手势和行为。理解 modifier 的顺序和可用 API 对正确性和性能至关重要。

## Modifier 链式顺序

顺序很重要。Modifier 在 DSL 中从左到右应用，但概念上是从下到上包裹。每个 modifier 接收一个绘制/测量其下方内容的 lambda。

```kotlin
// 示例：不同顺序产生不同结果
Box(
    Modifier
        .background(Color.Red)
        .padding(16.dp)
        .size(100.dp)
)
// 红色背景包裹带内边距的内容，内容再包裹 100x100 的 Box

Box(
    Modifier
        .size(100.dp)
        .padding(16.dp)
        .background(Color.Red)
)
// 100x100 的 Box 先加内边距，然后整体（132x132）获得红色背景
```

**应做：** 从外层（布局/尺寸）到内层（样式/交互）排列 modifier。
**不应做：** 如果希望内边距包含在最终尺寸中，不要将 `size` 放在 `padding` 之后。

来源：`compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/Modifier.kt`

## 常见 Modifier 模式

### Padding 和 Sizing

```kotlin
// Padding：内容周围的外边距
Box(Modifier.padding(16.dp)) { }

// Size：精确尺寸（覆盖父组件请求的尺寸）
Box(Modifier.size(100.dp)) { }
Box(Modifier.size(width = 200.dp, height = 100.dp)) { }

// FillMaxWidth/FillMaxHeight：扩展至可用空间
Box(Modifier.fillMaxWidth(0.8f)) { }  // 父宽度的 80%
Box(Modifier.fillMaxSize()) { }       // 父组件的 100%

// 应做：先 fillMaxWidth 再加 padding，对齐更清晰
Column(Modifier.fillMaxWidth()) {
    Box(Modifier.padding(16.dp).fillMaxWidth()) { }
}

// 不应做：如果希望背景扩展，不要在 background 之后应用 fillMaxWidth
// 正确做法：
Box(Modifier.fillMaxWidth().background(Color.Blue)) { }
```

### Background 和 Border

```kotlin
// Background 为表面应用颜色
Box(Modifier.background(Color.Blue)) { }
Box(Modifier.background(Color.Blue, shape = RoundedCornerShape(8.dp))) { }

// Border 绘制描边（顺序很重要！）
Box(
    Modifier
        .size(100.dp)
        .border(2.dp, Color.Black, RoundedCornerShape(8.dp))
        .background(Color.White)
)
// Border 在视觉顺序上在 background 之后绘制（因为下方的 modifier 先绘制）

// 组合 background + border：在链中先应用 border
Box(
    Modifier
        .border(2.dp, Color.Black, RoundedCornerShape(8.dp))
        .background(Color.White)
)
```

### Clipping

```kotlin
// 将内容裁剪为指定形状
Box(Modifier.clip(RoundedCornerShape(8.dp))) {
    Image(painter = painterResource(id = R.drawable.my_image), contentDescription = "")
}

// 应做：如果希望 background 在形状内部，先应用 clip
Box(
    Modifier
        .clip(RoundedCornerShape(8.dp))
        .background(Color.Blue)
) { }

// 不应做：先 background 再 clip（能工作但语义上不正确）
Box(
    Modifier
        .background(Color.Blue)
        .clip(RoundedCornerShape(8.dp))
) { }
```

## Clickable 和 Combined Clickable

```kotlin
// 基本点击处理，带涟漪效果（Material 3 默认）
Button(onClick = { }) { Text("Click me") }

// 手动 clickable，带涟漪
Box(
    Modifier
        .size(100.dp)
        .clickable(
            indication = ripple(),  // Material 涟漪反馈
            interactionSource = remember { MutableInteractionSource() }
        ) { /* 处理点击 */ }
)

// Combined clickable：长按 + 双击 + 单击
Box(
    Modifier
        .combinedClickable(
            onClick = { },
            onLongClick = { },
            onDoubleClick = { },
            indication = ripple()
        )
) { }

// 应做：为测试/状态观察提供显式 interactionSource
val interactionSource = remember { MutableInteractionSource() }
Box(
    Modifier.clickable(
        interactionSource = interactionSource,
        indication = ripple()
    ) { }
)

// 不应做：忘记 indication 参数（将没有视觉反馈）
Box(Modifier.clickable { }) { }  // 无涟漪
```

## Modifier.composed 与 Modifier.Node

旧 API（`composed`）正在被淘汰，推荐使用新的 `ModifierNodeElement` API。两者都能工作，但新代码应使用后者。

### 旧 API：Modifier.composed

```kotlin
fun Modifier.myCustomModifier(value: String) = composed {
    val state = remember { mutableStateOf(value) }
    this.then(
        Modifier
            .background(Color.Blue)
            .clickable { state.value = "updated" }
    )
}
```

- 创建新的 composable 作用域
- 捕获 composition locals
- remember state 变化时触发重组
- 已废弃但仍受支持

### 新 API：Modifier.Node

```kotlin
class MyCustomNode(val value: String) : Modifier.Node {
    override fun onDetach() {
        // 移除时清理
    }
}

data class MyCustomElement(val value: String) : ModifierNodeElement<MyCustomNode>() {
    override fun create() = MyCustomNode(value)
    override fun update(node: MyCustomNode) {
        node.value = value
    }
}

fun Modifier.myCustomModifier(value: String) = this.then(MyCustomElement(value))
```

**应做：** 为新的自定义 modifier 使用 `Modifier.Node`。它更高效且不创建 composition 作用域。
**不应做：** 创建新的 `composed` modifier；将现有的迁移到 `Modifier.Node`。

来源：`compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/modifier/ModifierNodeElement.kt`

## Layout vs Drawing vs Pointer Input Modifier

Modifier 分为不同类别，影响执行时机：

```kotlin
// Layout modifier：影响测量和布局阶段
fun Modifier.customSize(width: Dp, height: Dp) =
    this.then(object : LayoutModifier {
        override fun MeasureScope.measure(measurable: Measurable, constraints: Constraints) =
            measurable.measure(Constraints.fixed(width.roundToPx(), height.roundToPx()))
                .run { layout(width = size.width, height = size.height) { place(0, 0) } }
    })

// Drawing modifier：不影响布局，只在内容后绘制
fun Modifier.customDraw() = drawBehind { drawCircle(Color.Red) }

// Pointer input modifier：处理手势/事件
fun Modifier.detectCustomGesture() = pointerInput(Unit) {
    detectTapGestures { offset -> /* 处理 */ }
}
```

**应做：** 布局 modifier 用于尺寸/定位，绘制 modifier 用于视觉效果，指针 modifier 用于输入。
**不应做：** 使用布局 modifier 创建视觉效果；应使用绘制 modifier。

## Modifier.graphicsLayer — 性能影响

`graphicsLayer` 在图形渲染层面应用变换。对于动画来说，它比重组更高效。

```kotlin
// 高效：在 graphics layer 应用变换，无重组
Box(
    Modifier.graphicsLayer(
        scaleX = 1.2f,
        scaleY = 1.2f,
        translationX = 10f,
        rotationZ = 45f,
        alpha = 0.8f
    )
) { }

// 低效：每帧重组
var scaleX by remember { mutableStateOf(1f) }
LaunchedEffect(Unit) {
    while (true) {
        scaleX = 1.2f
        delay(16)
    }
}
Box(Modifier.scale(scaleX)) { }
```

**应做：** 为动画和频繁属性变化使用 `graphicsLayer`。
**不应做：** 在 `graphicsLayer` 足够时动画会触发重组的 state 值。

来源：`compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/graphics/GraphicsLayerModifier.kt`

## Modifier.testTag — UI 测试

```kotlin
// 添加 test tag 以便在测试中找到 composable
Box(Modifier.testTag("my_box")) { }

// 在测试中：
composeTestRule.onNodeWithTag("my_box").performClick()
composeTestRule.onNodeWithTag("my_box").assertIsDisplayed()
```

**应做：** 使用唯一、描述性的 test tag。
**不应做：** 在生产代码中将 test tag 用于业务逻辑。

## 审查清单：Modifier 顺序 Bug

### 调用方的 `modifier` 参数后硬编码尺寸

当 composable 接受 `modifier: Modifier = Modifier` 并在其后链式连接固定的 `.height()` / `.width()` / `.size()` 时，调用方的尺寸约束会被静默忽略或截断。

```kotlin
// BAD：调用方的高度是外层约束，组件的 172.dp 是内层 — 组件始终渲染为 172dp
@Composable
fun BannerCard(
    modifier: Modifier = Modifier,
) {
    Box(
        modifier = modifier          // 调用方约束先应用（外层）
            .fillMaxWidth()
            .height(172.dp)          // 内层 — 更小时胜出，更大时截断
            .clip(RoundedCornerShape(18.dp))
            .background(Color.Green.copy(alpha = 0.08f)),
    )
}

// 调用方期望 200dp 但得到 172dp：
BannerCard(modifier = Modifier.height(200.dp))

// 调用方期望 100dp — 组件被截断/压缩：
BannerCard(modifier = Modifier.height(100.dp))
```

**原因：** Modifier 链从外到内（左到右）解析。外层约束设置最大边界，内层约束在边界内请求。第一个尺寸约束作为上限胜出。

**修复方案 1：** 组件先设置默认值，调用方可通过 `.then(modifier)` 覆盖：
```kotlin
// GOOD：组件设置默认值，调用方的 modifier 最后应用并可覆盖
@Composable
fun BannerCard(
    modifier: Modifier = Modifier,
) {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(172.dp)
            .clip(RoundedCornerShape(18.dp))
            .background(Color.Green.copy(alpha = 0.08f))
            .then(modifier),         // 调用方可覆盖尺寸
    )
}
```

**修复方案 2：** 使用 `defaultMinSize` 实现灵活尺寸：
```kotlin
// GOOD：保证最小值，调用方可使其更大
@Composable
fun BannerCard(
    modifier: Modifier = Modifier,
) {
    Box(
        modifier = modifier
            .fillMaxWidth()
            .defaultMinSize(minHeight = 172.dp)  // 下限，不是上限
            .clip(RoundedCornerShape(18.dp))
            .background(Color.Green.copy(alpha = 0.08f)),
    )
}
```

**规则：** 当 composable 接受 `modifier: Modifier = Modifier` 时，永远不要在调用方的 modifier 之后链式连接固定的 `.height()` / `.width()` / `.size()` — 调用方约束成为外层边界，组件固定尺寸要么被忽略要么被截断。使用末尾的 `.then(modifier)` 或 `defaultMinSize` 实现灵活尺寸。在每次 PR 审查中标记此问题。

---

## 反模式

### 在 Composition 中创建 Modifier

```kotlin
// 不应做：每次重组都创建新 Modifier
@Composable
fun BadModifier() {
    Box(Modifier.padding(16.dp).background(Color.Blue)) { }
}

// 应做：提取为变量或参数
@Composable
fun GoodModifier(modifier: Modifier = Modifier) {
    Box(modifier.padding(16.dp).background(Color.Blue)) { }
}
```

### 错误的条件 Modifier 链

```kotlin
// 不应做：破坏类型检查和可读性
val mod = if (isSelected) Modifier.background(Color.Blue) else Modifier
Box(mod.padding(16.dp)) { }

// 应做：使用 then() 进行条件链式连接
Box(
    Modifier
        .padding(16.dp)
        .then(if (isSelected) Modifier.background(Color.Blue) else Modifier)
) { }
```

---

**总结：** 掌握 modifier 顺序，优先使用 `Modifier.Node` 而非 `composed`，为动画使用 `graphicsLayer`。
