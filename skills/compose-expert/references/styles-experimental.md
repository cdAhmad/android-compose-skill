# Compose Styles API（实验性）

> `@ExperimentalFoundationStyleApi` — `androidx.compose.foundation:foundation:1.11.0-alpha06`
>
> AOSP 来源：`compose/foundation/foundation/src/commonMain/kotlin/androidx/compose/foundation/style/`

## 什么是 Styles API？

Compose 的声明式、状态驱动的样式系统。无需为每个交互状态手动链式连接 modifier
和 `animateXAsState` 调用，你只需在单个 `Style { }` 块中声明所有视觉状态。框架自动处理
状态检测、属性插值和动画。

### 前后对比

```kotlin
// 之前 — 15+ 行命令式状态连接
val interactionSource = remember { MutableInteractionSource() }
val isPressed by interactionSource.collectIsPressedAsState()
val bgColor by animateColorAsState(
    if (isPressed) Color.DarkBlue else Color.Blue
)
val scale by animateFloatAsState(
    if (isPressed) 0.95f else 1f
)
Box(
    Modifier
        .graphicsLayer { scaleX = scale; scaleY = scale }
        .background(bgColor, RoundedCornerShape(16.dp))
        .clip(RoundedCornerShape(16.dp))
        .clickable(interactionSource = interactionSource, indication = null) { }
        .padding(16.dp)
)

// 之后 — 声明式。搞定。
val style = Style {
    background(Color.Blue)
    shape(RoundedCornerShape(16.dp))
    contentPadding(16.dp)
    pressed(Style {
        animate(Style {
            background(Color.DarkBlue)
            scale(0.95f)
        })
    })
}
Box(Modifier.styleable(styleState = styleState, style = style))
```

思维转变：**停止告诉 Compose 如何在状态之间动画**。声明每个状态
长什么样 — 框架自动插值。

---

## 三个组成部分

### 1. `Style { }` — 声明视觉状态

`Style` 是一个 `fun interface`。使用构建器 DSL：

```kotlin
val cardStyle = Style {
    // 基础属性（始终应用）
    background(Color(0xFFF5F5F5))
    shape(RoundedCornerShape(12.dp))
    contentPadding(16.dp)

    // 状态覆盖 — 仅在状态激活时应用
    selected(Style {
        animate(Style {
            background(Color.Blue.copy(alpha = 0.15f))
            borderWidth(2.dp)
            borderColor(Color.Blue)
        })
    })

    disabled(Style {
        background(Color(0xFFE0E0E0))
        contentColor(Color.Gray)  // 无 animate = 立即 snap
    })
}
```

`animate(Style { })` 包装应平滑插值的属性。
没有 `animate`，状态变化立即 snap。

### 2. `MutableStyleState` — 驱动状态

```kotlin
// 用于切换状态（checked、selected、enabled）— 显式设置：
val styleState = remember { MutableStyleState(MutableInteractionSource()) }
styleState.isChecked = isChecked
styleState.isSelected = isSelected
styleState.isEnabled = isEnabled

// 用于交互状态（pressed、hovered、focused）— 共享 interactionSource：
val interactionSource = remember { MutableInteractionSource() }
val styleState = remember { MutableStyleState(interactionSource) }
// isPressed、isHovered、isFocused 从共享的 interactionSource 自动跟踪
```

### 3. `Modifier.styleable()` — 应用于任何 Composable

```kotlin
Box(
    Modifier
        .styleable(styleState = styleState, style = cardStyle)
        .clickable(interactionSource = interactionSource, indication = null) { }
)
```

背景渲染、形状裁剪、边框绘制、变换应用、文本属性通过
CompositionLocal 传播给子组件。全部带动画。

---

## 关键：Alpha06 自动检测已损坏

**`styleable(style = myStyle)` 不带显式 `styleState` 不会检测来自同级 modifier 的交互
状态。** 这是 alpha06 中最大的陷阱。

### 不起作用的情况：

```kotlin
// 能编译。渲染基础样式。状态变化静默失效。
Box(
    Modifier
        .styleable(style = myStyle)           // 没有 styleState！
        .toggleable(value = isChecked, ...)   // 样式永远看不到这个
)
```

### 起作用的情况：

**模式 A — 切换状态（checked、selected、enabled）：**
```kotlin
val styleState = remember { MutableStyleState(MutableInteractionSource()) }
styleState.isChecked = isChecked  // 由你驱动状态

Box(
    Modifier
        .styleable(styleState = styleState, style = myStyle)
        .clickable { isChecked = !isChecked }
)
```

**模式 B — 交互状态（pressed、hovered、focused）：**
```kotlin
val interactionSource = remember { MutableInteractionSource() }
val styleState = remember { MutableStyleState(interactionSource) }

Box(
    Modifier
        .styleable(styleState = styleState, style = myStyle)
        .clickable(
            interactionSource = interactionSource,  // 同一个实例！
            indication = null,
        ) { }
)
```

**模式 C — 两者都有（带按下反馈的切换按钮）：**
```kotlin
val interactionSource = remember { MutableInteractionSource() }
val styleState = remember { MutableStyleState(interactionSource) }
styleState.isChecked = isChecked  // 显式设置切换状态

Box(
    Modifier
        .styleable(styleState = styleState, style = myStyle)
        .clickable(
            interactionSource = interactionSource,
            indication = null,
        ) { isChecked = !isChecked }
)
```

**规则：始终将 `styleState` 传递给 `styleable()`。**

---

## 文本属性传播陷阱

`contentColor()`、`fontSize()`、`fontWeight()`、`letterSpacing()`、`textDecoration()` 和
其他文本属性通过 `CompositionLocal`（`LocalContentColor`、`LocalTextStyle`）
传播到 styleable box 内部的所有子 composable。

### 问题：
```kotlin
// Style 中的 fontSize(28.sp) 应用于两个文本！
Box(Modifier.styleable(style = Style { fontSize(28.sp) })) {
    Text("Title")      // 28sp
    Text("Subtitle")   // 也是 28sp — 重叠！
}
```

### 修复：
在 styled box 中只使用单个 `Text`。将描述移出 styleable 作用域：

```kotlin
Text("Description goes here")  // 在 styled box 外部
Box(Modifier.styleable(style = gradientStyle)) {
    Text("Title Only")  // 只有这个获得样式
}
```

---

## 已验证属性（alpha06，设备上测试）

| 属性 | 是否工作？ | 说明 |
|----------|--------|-------|
| `background(Color)` | 是 | 在内容后方填充 |
| `background(Brush)` | 是 | 渐变背景 |
| `shape(Shape)` | 是 | 裁剪内容 + 背景 |
| `contentPadding(Dp)` | 是 | 内边距 |
| `borderWidth(Dp) + borderColor(Color)` | 是 | 必须同时设置两者 |
| `scale(Float)` | 是 | graphicsLayer 变换 |
| `rotationZ(Float)` | 是 | graphicsLayer 旋转 |
| `translationX/Y(Float)` | 是 | graphicsLayer 偏移 |
| `alpha(Float)` | 是 | 不透明度 |
| `contentColor(Color)` | 是 | 传播给子 Text/Icon |
| `contentBrush(Brush)` | 是 | 渐变文本 |
| `fontSize(TextUnit)` | 是 | 传播给子组件 |
| `fontWeight(FontWeight)` | 是 | 传播给子组件 |
| `letterSpacing(TextUnit)` | 是 | 传播给子组件 |
| `textDecoration(TextDecoration)` | 是 | 下划线、删除线 |
| `animate(Style { })` | 是 | 平滑弹簧插值 |
| `dropShadow(Shadow)` | 否 | `Shadow` 构造函数是 internal |

---

## Style 组合

Styles 通过 `.then()` 组合 — 后面的 style 按属性覆盖前面的：

```kotlin
val base = Style {
    background(Color.Blue)
    shape(RoundedCornerShape(12.dp))
    contentPadding(16.dp)
}

val elevated = Style {
    borderWidth(2.dp)
    borderColor(Color.LightGray)
    scale(1.02f)
}

val dark = Style {
    background(Color(0xFF1E1E2E))  // 覆盖 base 的 background
    contentColor(Color.White)
}

// 链式：
val composed = base.then(elevated).then(dark)

// 工厂（等价）：
val composed = Style(base, elevated, dark)
```

---

## 构建可复用组件

Styles API 映射到 Compose 的组件约定。`style` 参数成为
一等公民，类似于 `modifier`：

```kotlin
// 1. Defaults 对象 — 主题感知的默认样式
@OptIn(ExperimentalFoundationStyleApi::class)
object StyledChipDefaults {
    @Composable
    fun style(): Style {
        val bg = MaterialTheme.colorScheme.secondaryContainer
        val fg = MaterialTheme.colorScheme.onSecondaryContainer
        return Style {
            background(bg)
            shape(RoundedCornerShape(8.dp))
            contentPadding(horizontal = 16.dp, vertical = 8.dp)
            contentColor(fg)
            pressed(Style {
                animate(Style { scale(0.95f) })
            })
        }
    }
}

// 2. 组件 — style 作为带默认值的参数
@Composable
fun StyledChip(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    style: Style = StyledChipDefaults.style(),
    content: @Composable () -> Unit,
) {
    val interactionSource = remember { MutableInteractionSource() }
    val styleState = remember { MutableStyleState(interactionSource) }

    Box(
        modifier = modifier
            .styleable(styleState = styleState, style = style)
            .clickable(
                interactionSource = interactionSource,
                indication = null,
                onClick = onClick,
            ),
        contentAlignment = Alignment.Center,
    ) {
        content()
    }
}

// 3. 使用 — 默认、自定义或组合
StyledChip(onClick = {}) { Text("Default") }

StyledChip(
    onClick = {},
    style = Style {
        background(Color.Teal)
        shape(CircleShape)
        contentColor(Color.Black)
    },
) { Text("Custom") }

StyledChip(
    onClick = {},
    style = StyledChipDefaults.style().then(Style {
        borderWidth(2.dp)
        borderColor(Color.Teal)
    }),
) { Text("Composed") }
```

---

## 主题集成

`StyleScope` 扩展 `CompositionLocalAccessorScope`，因此 Style 块可以在解析时读取
`MaterialTheme` 值：

```kotlin
@Composable
fun ThemedButton() {
    val primary = MaterialTheme.colorScheme.primary
    val onPrimary = MaterialTheme.colorScheme.onPrimary
    val surface = MaterialTheme.colorScheme.surface

    val style = Style {
        background(primary)
        contentColor(onPrimary)
        shape(RoundedCornerShape(12.dp))
        contentPadding(16.dp)
        pressed(Style {
            animate(Style {
                background(surface)
                contentColor(primary)
                scale(0.95f)
            })
        })
    }
    // 主题变化（暗/亮）时，样式自动重新解析
}
```

在 `@Composable` 作用域中捕获主题颜色，在 Style 构建器中使用。主题切换
立即更新所有 styled 元素。

---

## 架构：工作原理

API 位于 `androidx.compose.foundation.style` 下的 7 个源文件中：

| 文件 | 用途 |
|------|---------|
| `Style.kt` | `fun interface Style` + 组合操作符 |
| `StyleScope.kt` | ~50 个属性函数 |
| `StyleState.kt` | `StyleState` 接口 + `MutableStyleState` |
| `StyleModifier.kt` | `Modifier.styleable()` 实现 |
| `StyleAnimations.kt` | `animate()` 块 |
| `ResolvedStyle.kt` | 带位集标记的属性解析 |
| `ExperimentalFoundationStyleApi.kt` | Opt-in 注解 |

### 双节点系统

`Modifier.styleable()` 插入两个 modifier 节点：

- **`StyleOuterNode`** — 布局（padding、sizing）、绘制（background、border、shape）、
  变换（scale、rotation、translation、alpha）。当变换/绘制属性变化时，可以仅在绘制层使无效 — 无重组。

- **`StyleInnerNode`** — 内容 padding 和文本样式传播。设置 `LocalContentColor`、
  `LocalTextStyle` 等，以便子 `Text` 和 `Icon` composable 获取 styled 颜色/字体。

### 基于位集的属性跟踪

`ResolvedStyle` 使用位集标记处理 ~50 个属性。状态变化时：

1. 仅计算旧和新解析属性之间的增量
2. 仅绘制变化（background、border、alpha） → **仅绘制无效化**（跳过布局 + 组合）
3. 布局变化（padding、sizing） → 布局无效化
4. 文本变化（contentColor、fontSize） → 组合无效化（更新 CompositionLocals）

仅改变 `scale` 和 `background` 的按下动画永远不会触发重组。

---

## 所有 StyleScope 属性

### 布局
- `contentPadding(Dp)`、`contentPadding(horizontal, vertical)`、`contentPadding(start, top, end, bottom)`
- `externalPadding(Dp)` 及相同变体
- `width(Dp)`、`height(Dp)`、`size(Dp)`、`size(width, height)`
- `minWidth/minHeight/maxWidth/maxHeight(Dp)`
- `fillWidth()`、`fillHeight()`、`fillSize()`

### 绘制
- `background(Color)`、`background(Brush)`
- `foreground(Color)`、`foreground(Brush)`
- `shape(Shape)`
- `borderWidth(Dp)`、`borderColor(Color)`、`borderBrush(Brush)`
- `border(width, color)`、`border(width, brush)`

### 变换
- `scale(Float)`、`scaleX(Float)`、`scaleY(Float)`
- `rotationX(Float)`、`rotationY(Float)`、`rotationZ(Float)`
- `translationX(Float)`、`translationY(Float)`、`translation(x, y)`
- `alpha(Float)`、`clip(Boolean)`、`zIndex(Float)`
- `transformOrigin(TransformOrigin)`

### 文本与内容
- `contentColor(Color)`、`contentBrush(Brush)`
- `fontSize(TextUnit)`、`fontWeight(FontWeight)`、`fontStyle(FontStyle)`
- `letterSpacing(TextUnit)`、`lineHeight(TextUnit)`
- `textDecoration(TextDecoration)`、`fontFamily(FontFamily)`
- `textAlign(TextAlign)`、`textDirection(TextDirection)`
- `textStyle(TextStyle)`、`textIndent(TextIndent)`
- `baselineShift(BaselineShift)`、`lineBreak(LineBreak)`
- `hyphens(Hyphens)`、`fontSynthesis(FontSynthesis)`

### 阴影（alpha06 中构造函数为 internal）
- `dropShadow(Shadow)`、`innerShadow(Shadow)`

### 状态函数
- `pressed(Style)`、`hovered(Style)`、`focused(Style)`
- `selected(Style)`、`checked(Style)`、`disabled(Style)`

### 动画
- `animate(Style)` — 默认弹簧
- `animate(spec: AnimationSpec<Float>, Style)` — 自定义 spec
- `animate(toSpec, fromSpec, Style)` — 非对称进入/退出

### 组合
- `Style.then(other: Style)` — 链式（后面的覆盖）
- `Style(style1, style2)` — 合并工厂
- `Style(vararg styles)` — 合并多个

---

## 常见陷阱

1. **忘记 `styleState`** — #1 bug。样式渲染但从不响应状态。
2. **未共享 `interactionSource`** — 没有它 pressed/hovered/focused 不会跟踪。
3. **styled box 中有多个 Text 子组件** — 全部继承 fontSize/fontWeight/contentColor。
4. **使用 `toggleable()` / `selectable()`** — 它们在内部创建自己的 interactionSource。使用 `.clickable()` 并显式在 `MutableStyleState` 上设置状态。
5. **缺少 `@OptIn(ExperimentalFoundationStyleApi::class)`** — 所有用法都需要。
6. **尝试使用 `dropShadow()`** — `Shadow` 构造函数在 alpha06 中是 internal，无法编译。
7. **clickable 上没有 `indication = null`** — 没有它你会在 styled 反馈之上获得默认涟漪。
