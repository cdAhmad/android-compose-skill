---
name: compose-expert
description: >
  Jetpack Compose Android UI 开发专家 skill。涵盖状态管理、视图组合、动画、导航、性能、
  设计稿转代码工作流以及线上崩溃模式。基于 androidx/androidx 真实源码分析。
  当用户提到 Compose、@Composable、remember、LaunchedEffect、Scaffold、NavHost、
  MaterialTheme、LazyColumn、Modifier、recomposition、Style、styleable、MutableStyleState
  或任何 Compose API 时触发。当用户说 "Android UI"、"Kotlin UI"、"compose layout"、
  "compose navigation"、"compose animation"、"material3"、"compose styles"、
  "design to compose"、"build this UI"、"implement this design"、
  "review this PR"、"review this code"、"check this diff"、
  或任何 GitHub PR URL (github.com/.*/pull/)、"design system"、"component library"、
  "atomic"、"reusable component"、"design tokens"、"atoms"、"molecules"、
  或询问现代 Kotlin UI 开发模式时触发。即使是随意提及如
  "my compose screen is slow" 或 "how do I pass data between screens"
  也应触发此 skill。
  同时于 session_start 时自动检测 Compose 项目 —— 详见 references/auto-init.md。
version: 2.1.2
---

> **安装提示：** 本 skill 现在以插件形式分发。
> 如果你手动将文件复制到 `~/.claude/skills/`，则处于未维护的安装路径，
> 不会收到更新。请通过以下方式迁移：
>
>     /plugin marketplace add aldefy/compose-skill
>     /plugin install compose-expert
>
> 有关 Codex 和 Copilot CLI 的迁移说明，请参阅 [MIGRATION.md](../docs/MIGRATION.md)。
> 此提示将保留至 v2.x 版本，并在 v3.0 时升级。

# Compose 专家 Skill

提供务实、不带有强烈主观意见的 Jetpack Compose 正确且高性能代码编写指南。
基于 `androidx/androidx` 真实源码分析。

## 审查模式

**激活条件：** 输入包含 GitHub PR URL（`github.com/.+/pull/\d+`）或
明确的审查短语："review this PR"、"review this diff"、"check this code"、
"what's wrong with this"。

审查模式激活时：
1. **不要**遵循下方的生成工作流
2. 阅读 `references/pr-review.md` 并严格遵循其工作流
3. 输出结构化的本地审查报告 —— 不要发布到 GitHub

## 工作流

帮助编写 Compose 代码时，遵循以下检查清单：

### 1. 理解需求
- 涉及哪个 Compose 层级？（Runtime、UI、Foundation、Material3、Navigation）
- 是状态问题、布局问题、性能问题还是架构问题？

### 2. 分析设计（如提供了视觉参考）
- 如果用户分享了 Figma 画框、截图或设计规范，查阅 `references/design-to-compose.md`
- 使用 5 步方法论将设计分解为 composable 树
- 将设计 token 映射到 MaterialTheme，将间距映射到 CompositionLocals
- 识别动画需求并查阅 `references/animation.md` 获取配方

### 3. 查阅正确的参考文件
回答前从 `references/` 中阅读相关参考文件：

| 主题 | 参考文件 |
|------|---------|
| `@State`、`remember`、`mutableStateOf`、状态提升、`derivedStateOf`、`snapshotFlow` | `references/state-management.md` |
| 组合 composable 结构、slot、抽取、预览 | `references/view-composition.md` —— 设计系统结构另见 `references/atomic-design.md` |
| Modifier 顺序、自定义 modifier、`Modifier.Node` | `references/modifiers.md` |
| `LaunchedEffect`、`DisposableEffect`、`SideEffect`、`rememberCoroutineScope` | `references/side-effects.md` |
| `CompositionLocal`、`LocalContext`、`LocalDensity`、自定义 local | `references/composition-locals.md` |
| `LazyColumn`、`LazyRow`、`LazyGrid`、`Pager`、key、content types | `references/lists-scrolling.md` |
| `NavHost`、类型安全路由、深层链接、共享元素转场 | `references/navigation.md` |
| `animate*AsState`、`AnimatedVisibility`、`Crossfade`、转场动画 | `references/animation.md` —— M3 token 选择另见 `references/material3-motion.md` |
| `MaterialTheme`、`ColorScheme`、动态取色、`Typography`、shapes | `references/theming-material3.md` —— 动效见 `references/material3-motion.md`；设计 token 见 `references/atomic-design.md` |
| 重组跳过、稳定性、baseline profiles、基准测试 | `references/performance.md` |
| 已移除/替换的 API、旧版本 Compose 的迁移路径 | `references/deprecated-patterns.md` |
| **Styles API**（实验性）：`Style {}`、`MutableStyleState`、`Modifier.styleable()` | `references/styles-experimental.md` |
| Figma/截图分解、设计 token、间距、modifier 顺序 | `references/design-to-compose.md` |
| 线上崩溃模式、防御性编码、状态/性能规则 | `references/production-crash-playbook.md` |
| M3 动效 token、`MotionTokens`、`MotionScheme`、动画时长、缓动曲线 | `references/material3-motion.md` |
| PR URL、代码审查、"review this PR"、"what's wrong with this" | `references/pr-review.md` |
| Session start、项目检测 | `references/auto-init.md` |
| 原子设计、设计系统、可复用组件、组件库、设计 token | `references/atomic-design.md` |

### 4. 应用并验证
- 编写符合参考文件中模式的代码
- 标记用户现有代码中的任何反模式
- 建议最简正确方案 —— 不要过度工程化

### 4a. 组件构建模式
当请求涉及构建组件（渲染 UI 的 composable）时：
- 查阅 `references/atomic-design.md`
- 确定组件层级（atom、molecule、organism、template）
- 在搭建代码前应用第 5 节的 "Ask" 提示
- 确保组件满足 atom 契约（modifier、slot、token、默认值）

### 5. 引用源码
引用 Compose 内部实现时，指向确切的源文件：
```
// 参见：compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composer.kt
```

## 核心原则

1. **Compose 按三个阶段思考**：Composition → Layout → Drawing。每个阶段中的状态读取
   仅触发该阶段及后续阶段的工作。

2. **重组频繁且廉价** —— 但前提是帮助编译器跳过未更改的作用域。使用稳定类型，
   避免在 composable 体内分配对象。

3. **Modifier 顺序很重要**。`Modifier.padding(16.dp).background(Color.Red)` 与
   `Modifier.background(Color.Red).padding(16.dp)` 在视觉上不同。

4. **状态应尽可能放在低层**，仅在需要时才提升到更高层。不要仅仅因为可以就
   把所有东西放进 ViewModel。

5. **副作用用于桥接 Compose 的声明式世界与命令式 API**。为每个场景选择正确的工具 ——
   误用会导致难以追踪的 bug。

6. **Android 平台 API 并非通用**。`LocalContext`、`BackHandler` 和
   `dynamicColorScheme` 是 Android 特有的。跨模块共享代码时，
   将平台需求抽象到接口后面。

## 源码凭证

除指导文档外，本 skill 还打包了来自 `androidx/androidx`（分支：androidx-main）的
**真实源码**。当你需要验证内部实现原理，或用户要求"给我看实际实现"时，
从 `references/source-code/` 中阅读原始源码：

| 模块 | 源码参考 | 核心文件 |
|------|---------|---------|
| Runtime | `references/source-code/runtime-source.md` | Composer.kt、Recomposer.kt、State.kt、Effects.kt、CompositionLocal.kt、Remember.kt、SlotTable.kt、Snapshot.kt |
| UI | `references/source-code/ui-source.md` | AndroidCompositionLocals.android.kt、Modifier.kt、Layout.kt、LayoutNode.kt、ModifierNodeElement.kt、DrawModifier.kt |
| Foundation | `references/source-code/foundation-source.md` | LazyList.kt、LazyGrid.kt、BasicTextField.kt、Clickable.kt、Scrollable.kt、Pager.kt |
| Material3 | `references/source-code/material3-source.md` | MaterialTheme.kt、ColorScheme.kt、Button.kt、Scaffold.kt、TextField.kt、NavigationBar.kt |
| Navigation | `references/source-code/navigation-source.md` | NavHost.kt、ComposeNavigator.kt、NavGraphBuilder.kt、DialogNavigator.kt |

### 双层方法
1. **先读指导文档** —— 阅读主题相关参考（如 `references/state-management.md`）
2. **再深入源码** —— 如果用户需要凭证或你需要验证，从 `references/source-code/` 读取

### 源码树映射
```
androidx/androidx (分支: androidx-main)
├── compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/
├── compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/
├── compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/
├── compose/foundation/foundation/src/commonMain/kotlin/androidx/compose/foundation/
├── compose/material3/material3/src/commonMain/kotlin/androidx/compose/material3/
├── compose/navigation/navigation-compose/src/commonMain/kotlin/androidx/navigation/compose/

```
