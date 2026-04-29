# Material 3 Motion

来源：`compose/material3/material3/src/commonMain/kotlin/androidx/compose/material3/tokens/MotionTokens.kt`
以及 `compose/material3/material3/src/commonMain/kotlin/androidx/compose/material3/MotionScheme.kt`
位于 `androidx/androidx`（分支：`androidx-main`）

`MotionTokens`、`MotionScheme` 以及所有缓动常量均位于
`androidx.compose.material3` — 自 M3 1.2.0 起可用。

---

## 1. 两种 API，同一套体系

M3 提供两种方式应用动效：

| API | 使用场景 |
|-----|---------|
| **`MotionScheme`**（推荐） | 在应适配应用动效方案的组件内部使用 — 主题控制使用基于弹簧还是基于插值的 spec |
| **`MotionTokens` + `tween()`** | 当你需要显式控制 `tween()` / `keyframes {}` 且组件不关心主题动效时 |

新组件使用 `MotionScheme`。当调用方显式提供 `AnimationSpec` 参数，或在使用 `AnimatedVisibility`、`Crossfade` 或共享元素时使用 `MotionTokens`。

---

## 2. MotionScheme API（组件首选）

`MotionScheme` 与 `colorScheme`、`typography`、`shapes` 一样，是 `MaterialTheme` 的一部分。

```kotlin
// 通过 MaterialTheme 访问
val motionScheme = MaterialTheme.motionScheme

// 两种内置方案
MaterialTheme(motionScheme = MotionScheme.standard())    // 实用型 UI
MaterialTheme(motionScheme = MotionScheme.expressive())  // 突出型 UI（M3 推荐默认）
```

### Spec 函数

| 函数 | 使用场景 |
|----------|---------|
| `defaultSpatialSpec<T>()` | 布局变化、位置/尺寸过渡（空间型） |
| `fastSpatialSpec<T>()` | 快速空间过渡 |
| `slowSpatialSpec<T>()` | 刻意放慢的空间过渡 |
| `defaultEffectsSpec<T>()` | 透明度、颜色、非空间变化 |
| `fastEffectsSpec<T>()` | 快速透明度/颜色过渡 |
| `slowEffectsSpec<T>()` | 刻意放慢的透明度/颜色过渡 |

```kotlin
@Composable
fun AnimatedCard(expanded: Boolean) {
    val motionScheme = MaterialTheme.motionScheme

    // 尺寸变化 = 空间型
    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        animationSpec = motionScheme.defaultSpatialSpec(),
        label = "card-size"
    )

    // 颜色变化 = 效果型
    val color by animateColorAsState(
        targetValue = if (expanded) MaterialTheme.colorScheme.primaryContainer
                      else MaterialTheme.colorScheme.surface,
        animationSpec = motionScheme.defaultEffectsSpec(),
        label = "card-color"
    )
}
```

> **与 `tween()` 的关键区别**：`MotionScheme` 的 spec 在 `expressive()` 中默认基于弹簧。实际的 spec 类型（弹簧 vs 插值）由主题控制，而非组件。这意味着动效可以在不改代码的情况下适配应用的动效方案。

---

## 3. 时长 Token

在需要显式 `tween()` 控制时使用。所有值来源于 `MotionTokens.kt`
（从 Material Design spec v0_103 生成）。时长为 `Float` 类型 — 对 `tween(durationMillis = ...)` 使用 `.toInt()`。

| Token | 值 (ms) | 使用场景 |
|-------|-----------|---------|
| `MotionTokens.DurationShort1` | 50ms | 微交互 — 涟漪扩散、复选框勾选 |
| `MotionTokens.DurationShort2` | 100ms | 小元素出现/消失 |
| `MotionTokens.DurationShort3` | 150ms | 图标过渡、选择指示器 |
| `MotionTokens.DurationShort4` | 200ms | 工具提示出现、芯片选择 |
| `MotionTokens.DurationMedium1` | 250ms | FAB 展开、卡片状态变化 |
| `MotionTokens.DurationMedium2` | 300ms | **最常用** — 对话框、底栏、导航抽屉 |
| `MotionTokens.DurationMedium3` | 350ms | 展开组件过渡 |
| `MotionTokens.DurationMedium4` | 400ms | 页面级面板过渡 |
| `MotionTokens.DurationLong1` | 450ms | 复杂布局变化 |
| `MotionTokens.DurationLong2` | 500ms | 共享元素进入 |
| `MotionTokens.DurationLong3` | 550ms | 共享元素 — 大内容 |
| `MotionTokens.DurationLong4` | 600ms | 完整容器形变 |
| `MotionTokens.DurationExtraLong1` | 700ms | 仅用于全屏过渡 |
| `MotionTokens.DurationExtraLong2` | 800ms | 仅用于全屏过渡 |
| `MotionTokens.DurationExtraLong3` | 900ms | 仅用于全屏过渡 |
| `MotionTokens.DurationExtraLong4` | 1000ms | 仅用于全屏过渡 |

---

## 4. 缓动 Token

所有值来源于 `MotionTokens.kt`。通过 `MotionTokens.Easing*CubicBezier` 访问。

| Token | CubicBezierEasing(x1, y1, x2, y2) | 方向 | 使用场景 |
|-------|------------------------------------|-----------|---------|
| `MotionTokens.EasingEmphasizedDecelerateCubicBezier` | `(0.05f, 0.7f, 0.1f, 1.0f)` | 进入 | 元素进入屏幕 — 快速启动，轻柔落定 |
| `MotionTokens.EasingEmphasizedAccelerateCubicBezier` | `(0.3f, 0.0f, 0.8f, 0.15f)` | 退出 | 元素离开屏幕 — 缓慢启动，快速退出 |
| `MotionTokens.EasingEmphasizedCubicBezier` | `(0.2f, 0.0f, 0.0f, 1.0f)` | 两者 | 大多数 M3 组件过渡的默认选择 |
| `MotionTokens.EasingStandardDecelerateCubicBezier` | `(0.0f, 0.0f, 0.0f, 1.0f)` | 进入 | 简单进入 — 表现力不如 Emphasized |
| `MotionTokens.EasingStandardAccelerateCubicBezier` | `(0.3f, 0.0f, 1.0f, 1.0f)` | 退出 | 简单退出 |
| `MotionTokens.EasingStandardCubicBezier` | `(0.2f, 0.0f, 0.0f, 1.0f)` | 两者 | 简单状态变化 |
| `MotionTokens.EasingLinearCubicBezier` | `(0.0f, 0.0f, 1.0f, 1.0f)` | — | 仅用于循环/重复动画 |
| `MotionTokens.EasingLegacyCubicBezier` | `(0.4f, 0.0f, 0.2f, 1.0f)` | — | 等价于 `FastOutSlowInEasing` — 新代码中不要使用 |
| `MotionTokens.EasingLegacyAccelerateCubicBezier` | `(0.4f, 0.0f, 1.0f, 1.0f)` | — | 等价于 `FastOutLinearInEasing` — 新代码中不要使用 |
| `MotionTokens.EasingLegacyDecelerateCubicBezier` | `(0.0f, 0.0f, 0.2f, 1.0f)` | — | 等价于 `LinearOutSlowInEasing` — 新代码中不要使用 |

> **进入/退出规则（始终遵守）：** 进入 = 减速缓动（快速启动，轻柔落定）。
> 退出 = 加速缓动（缓慢启动，快速离开）。进入和退出永远不要使用相同的缓动。
> `Legacy*` token 等价于 M3 之前的命名常量 — 新代码中不要使用。

---

## 5. 在 Compose 动画 API 中使用 Token

### animate*AsState（优先使用 MotionScheme）

```kotlin
// 颜色 — 效果型 spec
val color by animateColorAsState(
    targetValue = if (selected) MaterialTheme.colorScheme.primary
                  else MaterialTheme.colorScheme.surface,
    animationSpec = MaterialTheme.motionScheme.defaultEffectsSpec(),
    label = "selection-color"
)

// 如果需要显式 tween：
val color by animateColorAsState(
    targetValue = targetColor,
    animationSpec = tween(
        durationMillis = MotionTokens.DurationShort4.toInt(),   // 200ms
        easing = MotionTokens.EasingStandardCubicBezier         // 状态变化，非进入/退出
    ),
    label = "color"
)
```

### AnimatedVisibility（非对称进入/退出）

进入和退出必须使用不同的时长和缓动 — 退出总是更快。

```kotlin
AnimatedVisibility(
    visible = visible,
    enter = fadeIn(
        animationSpec = tween(
            durationMillis = MotionTokens.DurationMedium2.toInt(),      // 300ms
            easing = MotionTokens.EasingEmphasizedDecelerateCubicBezier // 进入
        )
    ) + slideInVertically(
        animationSpec = tween(
            durationMillis = MotionTokens.DurationMedium2.toInt(),
            easing = MotionTokens.EasingEmphasizedDecelerateCubicBezier
        ),
        initialOffsetY = { it / 4 }
    ),
    exit = fadeOut(
        animationSpec = tween(
            durationMillis = MotionTokens.DurationShort4.toInt(),       // 200ms — 退出更快
            easing = MotionTokens.EasingEmphasizedAccelerateCubicBezier // 退出
        )
    ) + slideOutVertically(
        animationSpec = tween(
            durationMillis = MotionTokens.DurationShort4.toInt(),
            easing = MotionTokens.EasingEmphasizedAccelerateCubicBezier
        ),
        targetOffsetY = { it / 4 }
    )
) {
    content()
}
```

### updateTransition（多属性，共享 spec）

```kotlin
val transition = updateTransition(targetState = expanded, label = "card")

val elevation by transition.animateDp(
    transitionSpec = {
        tween(
            durationMillis = MotionTokens.DurationMedium1.toInt(),  // 250ms
            easing = MotionTokens.EasingEmphasizedCubicBezier
        )
    },
    label = "elevation"
) { isExpanded -> if (isExpanded) 8.dp else 0.dp }

val cornerRadius by transition.animateDp(
    transitionSpec = {
        tween(
            durationMillis = MotionTokens.DurationMedium1.toInt(),
            easing = MotionTokens.EasingEmphasizedCubicBezier
        )
    },
    label = "corner-radius"
) { isExpanded -> if (isExpanded) 0.dp else 12.dp }
```

### 共享元素过渡

共享元素跨越屏幕边界 — 使用 Long 范围。

```kotlin
Modifier.sharedElement(
    state = rememberSharedContentState(key = "hero-${item.id}"),
    animatedVisibilityScope = animatedVisibilityScope,
    boundsTransform = { _, _ ->
        tween(
            durationMillis = MotionTokens.DurationLong2.toInt(),    // 500ms
            easing = MotionTokens.EasingEmphasizedCubicBezier
        )
    }
)
```

---

## 6. 决策树

按顺序回答以下问题以选择合适的时长：

1. **微交互？**（涟漪、复选框勾选、切换滑块 snap）
   → `DurationShort1` (50ms) 或 `DurationShort2` (100ms)

2. **组件状态变化？**（按钮按下反馈、芯片选择、图标切换、标签指示器）
   → `DurationShort3` (150ms) 或 `DurationShort4` (200ms)

3. **容器变化？**（卡片展开、FAB 展开/收缩、菜单打开、工具提示）
   → `DurationMedium1` (250ms) 或 `DurationMedium2` (300ms) ← 最常用

4. **屏幕级元素？**（对话框进入、底栏滑动、搜索栏展开、导航抽屉）
   → `DurationMedium3` (350ms) 或 `DurationMedium4` (400ms)

5. **共享元素 / hero 过渡？**（图片或卡片从列表展开到详情页）
   → `DurationLong1` (450ms) 或 `DurationLong2` (500ms)

6. **全屏复杂形变？**（整个屏幕布局变化）
   → `DurationLong3`–`DurationExtraLong1` (550ms–700ms)

**缓动规则（始终应用）：**
- 元素到达 → `EasingEmphasizedDecelerateCubicBezier`
- 元素离开 → `EasingEmphasizedAccelerateCubicBezier`
- 元素状态变化（留在屏幕上） → `EasingEmphasizedCubicBezier`
- 循环/无限 → `EasingLinearCubicBezier`
- 对于主题感知的组件，优先使用 `MotionScheme` spec 而非手动缓动

---

## 7. 审查标记

代码审查中需要捕获的模式。另见 `references/pr-review.md` 类别 3。

| 代码中的模式 | 标记 | 修复 |
|----------------|------|-----|
| `tween(50)` | 硬编码时长 | `MotionTokens.DurationShort1.toInt()` |
| `tween(100)` | 硬编码时长 | `MotionTokens.DurationShort2.toInt()` |
| `tween(150)` | 硬编码时长 | `MotionTokens.DurationShort3.toInt()` |
| `tween(200)` | 硬编码时长 | `MotionTokens.DurationShort4.toInt()` |
| `tween(250)` | 硬编码时长 | `MotionTokens.DurationMedium1.toInt()` |
| `tween(300)` | 硬编码时长 | `MotionTokens.DurationMedium2.toInt()` |
| `tween(350)` | 硬编码时长 | `MotionTokens.DurationMedium3.toInt()` |
| `tween(400)` | 硬编码时长 | `MotionTokens.DurationMedium4.toInt()` |
| `tween(N)` 含任意整数字面量 | 硬编码时长 | 最近的 `MotionTokens.Duration*` token |
| `FastOutSlowInEasing` | M3 前缓动 | `MotionTokens.EasingEmphasizedCubicBezier` |
| `LinearOutSlowInEasing` | M3 前缓动 | `MotionTokens.EasingEmphasizedDecelerateCubicBezier` |
| `FastOutLinearInEasing` | M3 前缓动 | `MotionTokens.EasingEmphasizedAccelerateCubicBezier` |
| `animateColorAsState(target)` 无 `animationSpec` | 缺少 spec | `MaterialTheme.motionScheme.defaultEffectsSpec()` |
| `enter` 和 `exit` 使用相同缓动 | 错误配对 | 进入用 Decelerate，退出用 Accelerate |
| 非共享元素动画时长 > 600ms | 过慢 | 缩减到 `DurationLong1`–`DurationLong2` |
| 新组件使用显式 `tween()` 而非 `MotionScheme` | 不感知主题 | 使用 `MaterialTheme.motionScheme.defaultSpatialSpec()` / `defaultEffectsSpec()` |
