# PR 审查模式

激活条件：输入包含 GitHub PR URL（`github.com/.+/pull/\d+`）或明确的审查
短语："review this PR"、"review this diff"、"check this code"、"what's wrong with this"。

审查模式激活时：
1. **不要**遵循 `SKILL.md` 中的生成工作流
2. 仅遵循本文档
3. 输出结构化的本地报告 — 不要发布到 GitHub

---

## 审查工作流

### 步骤 1 — 获取 diff

```bash
gh pr diff <PR_URL>
```

记录所有变更的 `.kt` 文件。保存输出。

### 步骤 2 — 获取完整文件内容

对于每个变更的 `.kt` 文件，获取**完整文件** — 不只是 diff 行。

```bash
# 获取 PR 元数据
gh pr view <PR_URL> --json headRefName,headRepository \
  --jq '{branch: .headRefName, repo: .headRepository.nameWithOwner}'

# 获取完整文件（替换 {owner}、{repo}、{path}、{branch}）
gh api "repos/{owner}/{repo}/contents/{path}?ref={branch}" \
  --jq '.content' | base64 -d
```

**为什么完整文件很重要：** Diff 显示什么变了。完整文件显示 composable
实际长什么样 — 包括 `modifier` 参数是否存在，以及 modifier 链
如何跨多行构建。单行 modifier 模式如
`Row(modifier = Modifier.fillMaxWidth().padding(16.dp)) {` 在该行本身未变更时，
在仅 diff 视图中是不可见的。

### 步骤 3 — 扫描项目设置

按优先级顺序运行。无论找到什么 lint 配置，始终运行步骤 d。

**a. `.editorconfig`：**
```bash
cat .editorconfig 2>/dev/null || echo "not found"
```
记录：`[*.kt]` 下的缩进大小、最大行长度、尾随逗号规则。

**b. ktlint 配置：**
```bash
grep -A 20 "\[\*\\.kt\]" .editorconfig 2>/dev/null
cat .ktlint 2>/dev/null || echo "not found"
```

**c. detekt：**
```bash
find . -name "detekt.yml" -o -name "detekt-config.yml" 2>/dev/null | head -3
```
如果找到，阅读并记录复杂度、命名和样式规则。

**d. 推断代码库约定（始终运行）：**
```bash
# 查找 3–5 个不在 diff 中的现有 composable 文件
find . -name "*.kt" -not -path "*/build/*" | xargs grep -l "@Composable" | \
  grep -v "Test" | head -5
```
对每个文件，记录：
- Modifier 链式：每行一个 vs 构造器行内联
- Modifier 参数名：`modifier` vs `Modifier`（两者都有效 — 记录团队使用哪个）
- 单 slot composable 上的尾随 lambda vs 命名 `content =`
- 单参数调用上的命名参数用法

构建一个**项目画像**。用它抑制误报 — 标记偏离团队自身约定的现象，
而非偏离外部样式指南。

### 步骤 4 — 运行检查清单

根据下方全部 5 个类别评估每个变更的 composable。
使用步骤 2 中的**完整文件**，而非步骤 1 中的 diff。

### 步骤 5 — 输出报告

使用本文档末尾的格式。

---

## Compose 审查检查清单

### 类别 1：Modifier 规范

扫描每个变更的 `@Composable` 函数的完整文件。

- [ ] **存在 Modifier 参数。** 每个渲染 UI 的 `@Composable` 函数必须有
  `modifier: Modifier = Modifier` 参数。如果不存在则标记。
  例外：仅用作内部实现细节且无布局影响的 private composable。

- [ ] **Modifier 传递给根元素。** `modifier` 参数必须应用于函数体中最外层的布局 composable — 不能被忽略、不能应用于内部元素、
  不能用于同级元素。

- [ ] **Modifier 未被拆分。** `modifier` 不得拆分到两个同级元素上。
  一个根元素接收它。

- [ ] **Modifier 顺序遵循 paint model。** 由外向内工作：
  `size / fillMaxWidth` → `padding` → `background / border` → `clickable / pointerInput`。
  标记以下特定反转：
  - 当意图是 background 包裹内容时 `background()` 在 `padding()` 之前
    （`Modifier.padding(16.dp).background(Color.Red)` = background 包裹 padding 区域；
    `Modifier.background(Color.Red).padding(16.dp)` = background 不包含 padding 区域）
  - `clickable()` 在 `padding()` 之前 — 缩小有效触摸区域
  - `size()` 或 `fillMaxWidth()` 在 `padding()` 之后 — 尺寸约束不再包含 padding

- [ ] **单行 modifier 检查。** 即使 diff 中未变更，也阅读完整构造器行。
  验证即使 modifier 链完全内联书写时顺序也是正确的：
  `Row(modifier = Modifier.fillMaxWidth().clickable { }.padding(16.dp))` — 这是错误的顺序
  （`clickable` 在 `padding` 之前会缩小触摸区域）。

- [ ] **不要对同一调整同时使用 `padding` + `offset`。** `offset` 不影响布局；
  `padding` 影响。它们不可互换。

### 类别 2：重组

- [ ] **无不稳定参数类型** 且无稳定性注解：
  - 普通 `List<T>` — 编译器推断为不稳定。使用 `@Immutable` 数据类包装器、
    `ImmutableList<T>`（kotlinx-collections-immutable），或在调用点注解
  - `HashMap`、`MutableMap`、任何可变集合 — 不稳定
  - 无 `@Stable` 的非 `data` 类 — 编译器无法推断稳定性

- [ ] **内联创建的 Lambda 没有 `remember`。** 父组件每次重组时创建的新 lambda 实例
  会阻止子组件被跳过。
  ```kotlin
  // BAD — 每次父组件重组都产生新 lambda
  MyComposable(onClick = { doSomething() })

  // OK — 稳定引用
  val onClick = remember { { doSomething() } }
  MyComposable(onClick = onClick)
  ```
  例外：如果 Compose 编译器中启用了强跳过模式（`freeCompilerArgs +=
  "-P", "plugin:androidx.compose.compiler.plugins.kotlin:experimentalStrongSkipping=true"`），
  lambda 默认是稳定的。

- [ ] **`derivedStateOf {}` 用于计算值。** 标记在 composable 体内计算的状态读取值
  未使用 `remember { derivedStateOf { ... } }` 的情况：
  ```kotlin
  // BAD — 任何状态变化时都重新计算和重组
  val isValid = username.isNotEmpty() && password.length > 8

  // OK
  val isValid by remember { derivedStateOf { username.isNotEmpty() && password.length > 8 } }
  ```

- [ ] **`remember {}` key 正确。**
  - 缺少 key 但内部引用了输入变量 — 过期值 bug
  - 永不变化的 key（`remember(Unit)` 或 `remember(constant)`） — 实际上等同于无重新计算的 `remember {}`

### 类别 3：M3 动效

与 `references/material3-motion.md` 交叉引用以获取 token 值和缓动名称。

- [ ] **`tween()`、`spring()` 或 `keyframes {}` 中无硬编码整数字面量时长。** 标记任何
  纯整数字面量的 `tween(N)`。建议最近的 `MotionTokens.Duration*` token。

- [ ] **无 M3 前缓动常量：**
  - `FastOutSlowInEasing` → `MotionTokens.EasingEmphasizedCubicBezier`
  - `LinearOutSlowInEasing` → `MotionTokens.EasingEmphasizedDecelerateCubicBezier`
  - `FastOutLinearInEasing` → `MotionTokens.EasingEmphasizedAccelerateCubicBezier`

- [ ] **`animateColorAsState` 有 `animationSpec`。** 无 spec 的 `animateColorAsState(target)`
  使用默认弹簧，不适合颜色过渡。
  建议：`animationSpec = MaterialTheme.motionScheme.defaultEffectsSpec()`

- [ ] **`AnimatedVisibility` enter/exit 缓动不对称。** 进入必须使用 Decelerate 缓动；
  退出必须使用 Accelerate 缓动。两者使用相同缓动是错误的。

- [ ] **非共享元素动画时长不超过 600ms。** 标记超过 `DurationLong4` (600ms) 的时长
  除非动画是共享元素或全屏过渡。

- [ ] **新组件使用 `MotionScheme` 而非原始 `tween()`。** 不接受 `AnimationSpec`
  参数的新组件应使用 `MaterialTheme.motionScheme.defaultSpatialSpec()` /
  `defaultEffectsSpec()` 以感知主题动效。

### 类别 4：列表与 Key

- [ ] **`LazyColumn`、`LazyRow`、`LazyVerticalGrid`、
  `LazyHorizontalGrid` 中的每个 `items()` 调用都有 `key = {}`。** 缺少 key 会导致错误的动画和条目复用 bug。
  ```kotlin
  // BAD
  items(movies) { movie -> MovieCard(movie) }

  // OK
  items(movies, key = { it.id }) { movie -> MovieCard(movie) }
  ```

- [ ] **异构列表存在 `contentType = {}`。** 当懒加载列表渲染多于
  一种类型的 composable（例如 header + item）时，必须指定 `contentType` 以便 Compose
  正确复用组合节点。

- [ ] **`LazyColumn` 内部不直接嵌套 `LazyColumn`** 除非内部那个有固定高度。无边界的嵌套懒加载列表会在运行时抛出 `java.lang.IllegalStateException`。


### 类别 5：原子设计

与 `references/atomic-design.md` 交叉引用以获取 token 模式和命名规则。

- [ ] **composable 体内无硬编码 `Color(0xFF...)`。** 颜色必须来自
  `MaterialTheme.colorScheme.*` 或应用级品牌 token（`CompositionLocal`）。
  例外：`Color.Transparent`、`Color.Unspecified`、以及作为
  明确设计选择的 `Color.White`/`Color.Black` 是可接受的。

- [ ] **无硬编码 `fontSize`、`fontWeight` 或 `TextStyle(...)`。** 排版必须来自
  `MaterialTheme.typography.*`。标记主题定义文件外的任何内联 `TextStyle(fontSize = 14.sp)` 或
  `fontWeight = FontWeight.Bold`。

- [ ] **无魔法数字间距（`16.dp`）而无 token。** 如果项目定义了间距
  尺度（检查带有间距值的 `CompositionLocal`），标记应使用
  尺度的原始 `dp` 值。如果不存在间距尺度，将其记录为建议 — 不是关键问题。

- [ ] **Composable 名称描述功能，而非上下文。** 标记匹配以下
  模式的 composable：`*For*`（例如 `ButtonForSettings`）、`*With*`（例如 `CardWithRedBorder`）、
  `*In*`（例如 `HeaderInHome`）。例外：用于提供
  默认参数的 `*WithDefaults` 模式是可接受的。

- [ ] **Public composable 有 `modifier: Modifier = Modifier`。**（与类别 1 重叠 — 在
  原子设计上下文中，额外验证 modifier 是否传递给根元素且未被内部元素消耗。）

- [ ] **渲染可变内容的 composable 有 slot API。** 标记硬编码
  `Text("Submit")`、`Icon(Icons.Default.Close, ...)` 或类似固定内容的 composable，那些内容
  应作为 slot 参数。例外：设计上固定内容的内部/private composable。

- [ ] **Organism 不直接引用 ViewModel。** 任何组合多个
  UI 组件的 composable（organism 级别）必须接受数据和回调作为参数。标记 organism 内部直接的
  `viewModel()`、`hiltViewModel()` 或 `koinViewModel()` 调用。
  Screen 级别的 composable 才是访问 ViewModel 的正确位置。

---

## 输出报告格式

```
## PR Review: <PR title> (#NNN)
Branch: <head-branch> → <base-branch>

### Project Profile
- Code style: <推断 — 例如 "modifier 链每行一个"、"优先尾随 lambda">
- Lint config: <ktlint / detekt / 两者都未找到>
- Conventions from: <采样的文件>

---

### Issues

#### Critical
导致 bug、崩溃或正确性问题的 issue。

- `path/File.kt:42` — `MyCard` 缺少 `modifier: Modifier = Modifier` 参数。所有 UI
  composable 必须暴露 modifier 供调用方控制布局。
  修复：在签名中添加 `modifier: Modifier = Modifier`；传递给根元素。

#### Suggestions
样式、M3 对齐和性能改进。

- `path/File.kt:87` — `tween(300)` → `MotionTokens.DurationMedium2.toInt()` (300ms = Medium2)
- `path/File.kt:103` — `FastOutSlowInEasing` → `MotionTokens.EasingEmphasizedCubicBezier`
- `path/File.kt:115` — `items(movies)` 缺少 `key = { it.id }` — 添加 key 以防止重排 bug

#### Positive Patterns
良好的 Compose 用法 — 始终至少包含一条。

- `path/File.kt:55` — 正确的 `derivedStateOf {}` 防止冗余重组
- `path/File.kt:71` — `sharedBounds()` 正确用于容器到页面展开

---

### Summary
<N> 个 critical，<M> 个 suggestions，跨越 <K> 个审查文件。
```

**章节策略：**
- Critical 和 Suggestions 始终存在（为空时写 "None found"）
- Positive Patterns 始终存在 — 审查不能只读起来像一份打击清单
