<p align="center">
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/jetpackcompose/jetpackcompose-original.svg" width="80" alt="Jetpack Compose logo"/>
</p>

<h1 align="center">Compose Agent Skill</h1>

<p align="center">
  让你的 AI 编程工具真正理解 Android 上的 Jetpack Compose。<br/>
  基于 <code>androidx/androidx</code> 的真实源码 —— 不是凭感觉。<br/>
  Backed by real source code from <code>androidx/androidx</code> —— not vibes.
</p>

<p align="center">
  <a href="#安装"><img src="https://img.shields.io/badge/setup-5%20min-brightgreen" alt="Setup time"/></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue" alt="License"/></a>
  <a href="https://developer.android.com/jetpack/compose"><img src="https://img.shields.io/badge/Jetpack%20Compose-1.7+-4285F4" alt="Compose version"/></a>
  <a href="https://kotlinlang.org"><img src="https://img.shields.io/badge/Kotlin-2.0+-7F52FF" alt="Kotlin version"/></a>
</p>

---

## 安装

本 Skill 以插件形式分发。各宿主工具的安装方式请参考 **[docs/INSTALL.md](docs/INSTALL.md)**：

- **Claude Code:** `/plugin marketplace add aldefy/compose-skill` → `/plugin install compose-expert`
- **Copilot CLI:** `copilot plugin install aldefy/compose-skill`
- **Codex CLI:** 手动安装 —— 详见 INSTALL.md。

之前手动安装过？请参考 **[docs/MIGRATION.md](docs/MIGRATION.md)** 迁移。

## 更新

新版本通过 [GitHub Releases](https://github.com/aldefy/compose-skill/releases) 发布，并附带迁移说明。使用 `/plugin update`（Claude Code）或 `copilot plugin update aldefy/compose-skill`（Copilot）即可更新。

---

## 痛点

AI 编程工具生成的 Compose 代码虽然能编译，但细节往往不对。`remember` 用法错误、不必要的重组、Modifier 顺序写反、导航模式已经废弃、API 根本不存在……它们在猜测行为，而不是真正理解。

这个 Skill 通过两样东西解决上述问题：

1. **18 份参考指南** —— 涵盖 Compose 所有核心主题，包括 Material 3 动效、原子设计系统、设计稿转代码工作流、动画配方，以及线上崩溃模式
2. **5 份源码文件** —— 直接取自 [`androidx/androidx`](https://github.com/androidx/androidx/tree/androidx-main/compose)，让 AI 能在需要时查证真实实现

## 安装前后的区别

| 方面 | 没装 Skill | 装了 Skill |
|---|---|---|
| 状态管理 | 到处用 `remember { mutableStateOf() }`，该用 `derivedStateOf` 或 `rememberSaveable` 的时候也用错 | 根据场景选择正确的状态原语 |
| 性能 | 生成每一帧都重组的代码 | 使用稳定性注解、延迟读取、列表加 `key {}` |
| 导航 | 字符串路由（已废弃） | 基于 `@Serializable` 的类型安全路由 |
| Modifier | 顺序随意，看不出 `clickable` 写在 `padding` 前面的 bug | 按正确顺序书写，并能说明原因 |
| 副作用 | 协程作用域选错、`LaunchedEffect` 的 key 不对 | 选对副作用类型、key 与生命周期匹配 |
| API |  hallucinate 不存在的参数 | 先查真实源码再建议 |
| 设计稿转代码 | 字面翻译 Figma 节点、Modifier 顺序错误 | 使用语义化的 M3 组件、正确顺序、主题 token |
| 崩溃防护 | 没有防御性写法 | 防范零尺寸 DrawScope、重复 key、失效的 `derivedStateOf` 等 |

## 覆盖内容

| 主题 | AI 能学到什么 |
|---|---|
| 状态管理 | `remember`、`mutableStateOf`、`derivedStateOf`、`rememberSaveable`、状态提升、`snapshotFlow` |
| 视图组合 | Composable 结构规范、Slot API、`@Preview` 模式、抽取规则 |
| 性能 | 跳过重组、`@Stable`/`@Immutable`、延迟读取、Baseline Profile、基准测试 |
| 导航 | 类型安全路由、`NavHost`、深层链接、共享元素转场、返回栈 |
| 动画 | `animate*AsState`、`AnimatedVisibility`、`Crossfade`、`updateTransition`、共享转场、9 种配方（shimmer、滑动删除等）、手势驱动动画、Figma 曲线映射 |
| 列表与滚动 | `LazyColumn`/`LazyRow`/`LazyGrid`、`Pager`、`key {}`、`contentType`、滚动状态 |
| 副作用 | `LaunchedEffect`、`DisposableEffect`、`SideEffect`、`rememberCoroutineScope` |
| Modifier | 顺序规则、自定义 Modifier、`Modifier.Node` 迁移 |
| 主题 | `MaterialTheme`、`ColorScheme`、动态取色、`Typography`、形状、深色模式 |
| 无障碍 | Semantics、`contentDescription`、遍历顺序、触摸目标、TalkBack |
| CompositionLocal | `LocalContext`、`LocalDensity`、自定义 Local、何时用参数传递更合适 |
| 废弃模式 | 已移除的 API、旧版本 Compose 的迁移路径 |
| **Styles API（实验性）** | `Style {}`、`MutableStyleState`、`Modifier.styleable()`、组合、主题集成、alpha06 陷阱 |
| **设计稿转代码** | Composable 分解算法、Figma 属性映射、间距归属、Modifier 顺序、设计 token |
| **线上崩溃手册** | 6 种崩溃模式的根因 + 修复、防御性写法、生产环境状态/性能规则 |
| **M3 动效** | 全部时长 token（`DurationShort1–4`、`DurationMedium1–4`、`DurationLong1–4`、`DurationExtraLong1–4`）、缓动 token 与 CubicBezierEasing 值、`MotionScheme` API（`defaultSpatialSpec`、`defaultEffectsSpec`）、Compose API 映射、决策树、PR 审查标记 |
| **原子设计** | 5 层层级（token、atom、molecule、organism、template）映射到 Compose、M3 包装模式、自定义 atom 模式、Slot API 契约、token 层、反模式 |
| 源码 | `androidx/androidx` 中 runtime、UI、foundation、material3、navigation 的真实 `.kt` 文件 |

## 工作原理

```
你问 Compose 相关问题
        |
        v
  AI 读取 SKILL.md（工作流 + 检查清单）
        |
        v
  查询对应的参考文件
        |
        +-- state-management.md
        +-- performance.md
        +-- navigation.md
        +-- design-to-compose.md
        +-- production-crash-playbook.md
        +-- ...（共 19 份指南）
        |
        +-- source-code/
              +-- runtime-source.md
              +-- material3-source.md
              +-- ...（共 5 份源码文件）
```

**第一层：指导文档**（19 份）—— 包含模式、陷阱、正反例的实用参考。AI 优先读取这一层。

**第二层：源码凭证**（5 份）—— 来自 `androidx/androidx` 的真实 Kotlin 源码。当 AI 需要验证实现细节而非猜测时，读取这一层。

## 文件结构

```
skills/compose-expert/
├── SKILL.md                              # 主工作流 + 检查清单
└── references/
    ├── state-management.md               # 状态、remember、状态提升、derivedStateOf
    ├── view-composition.md               # Composable 结构规范、Slot、Preview
    ├── modifiers.md                      # Modifier 顺序、自定义 Modifier、Modifier.Node
    ├── side-effects.md                   # LaunchedEffect、DisposableEffect、SideEffect
    ├── composition-locals.md             # CompositionLocal、LocalContext、自定义 Local
    ├── lists-scrolling.md                # LazyColumn/Row/Grid、Pager、key、contentType
    ├── navigation.md                     # NavHost、类型安全路由、深层链接
    ├── animation.md                      # animate*AsState、AnimatedVisibility、转场
    ├── theming-material3.md              # MaterialTheme、ColorScheme、动态取色
    ├── performance.md                    # 重组、稳定性、基准测试
    ├── accessibility.md                  # Semantics、contentDescription、测试
    ├── deprecated-patterns.md            # 已移除 API、迁移路径
    ├── styles-experimental.md           # Styles API（@ExperimentalFoundationStyleApi）
    ├── design-to-compose.md             # Figma/截图分解、属性映射
    ├── production-crash-playbook.md     # 崩溃模式、防御性编码、生产规则
    ├── atomic-design.md                 # 原子设计系统：token、atom、molecule、organism、template
    └── source-code/                      # 真实 .kt 源码
        ├── runtime-source.md             # Composer、Recomposer、State、Effects
        ├── ui-source.md                  # AndroidCompositionLocals、Modifier、Layout
        ├── foundation-source.md          # LazyList、BasicTextField、Gestures
        ├── material3-source.md           # MaterialTheme、所有 M3 组件
        ├── navigation-source.md          # NavHost、ComposeNavigator
```

## 安装方式

Skill 本质就是 Markdown 文件。以下每种工具读取的内容完全一致，按需选择即可。

---

### Claude Code

Skill 以文件形式组织 —— Claude Code 会自动从 `~/.claude/skills/`（个人级）或 `.claude/skills/`（项目级）发现它们。

**个人 Skill（所有项目可用）：**
> 克隆仓库并复制到个人 skills 目录：

```bash
git clone https://github.com/aldefy/compose-skill.git /tmp/compose-skill
mkdir -p ~/.claude/skills
cp -r /tmp/compose-skill/skills/compose-expert ~/.claude/skills/
```

**项目级 Skill（仅当前项目）：**

> 克隆仓库并复制到项目的 `.claude/skills` 目录：

```bash
git clone https://github.com/aldefy/compose-skill.git /tmp/compose-skill
mkdir -p .claude/skills
cp -r /tmp/compose-skill/skills/compose-expert .claude/skills/
```

无需 CLI 命令或配置文件。Claude Code 会自动从这些目录读取 `SKILL.md`，并在你提到 Compose、`@Composable`、`remember`、`LazyColumn`、`NavHost` 等关键词时触发。

---

### Codex CLI (OpenAI)

在项目根目录添加 `AGENTS.md`：

```markdown
# AGENTS.md

## Jetpack Compose
所有 Compose/Android UI 任务，遵循 `skills/compose-expert/SKILL.md` 中的指令，
并在回答前查阅 `skills/compose-expert/references/` 中的参考文件。
```

将 Skill 作为子模块添加到项目：

```bash
git submodule add https://github.com/aldefy/compose-skill.git .compose-skill
```

Codex 会自动从 git 根目录到当前目录逐层发现 `AGENTS.md` 文件。

---

### Gemini CLI (Google)

在项目根目录添加 `GEMINI.md`：

```markdown
# GEMINI.md

## Jetpack Compose Expert
所有 Jetpack Compose 任务，遵循 `skills/compose-expert/SKILL.md` 中的工作流和检查清单。

回答任何 Compose 问题之前，先查阅对应参考文件：
- 状态管理 -> `skills/compose-expert/references/state-management.md`
- 性能 -> `skills/compose-expert/references/performance.md`
- 导航 -> `skills/compose-expert/references/navigation.md`
- （完整主题映射见 SKILL.md）

如需验证实现细节，查阅 `skills/compose-expert/references/source-code/` 中的源码。
```

作为子模块添加：

```bash
git submodule add git@github.com:aldefy/compose-skill.git .compose-skill
```

---

### Google Antigravity

Antigravity 会自动从工作区或全局 skill 文件夹发现 Skill。`skills/compose-expert` 目录本身就是一个完整的 Antigravity Skill，因为它包含带 YAML 前置描述的 `SKILL.md` 文件。

**工作区 Skill（项目级）：**

```bash
# 克隆仓库
git clone https://github.com/aldefy/compose-skill.git /tmp/compose-skill

# 复制到项目的 .agents/skills 目录
mkdir -p .agents/skills
cp -r /tmp/compose-skill/skills/compose-expert .agents/skills/compose-expert
```

**全局 Skill（所有项目可用）：**

```bash
# 克隆仓库
git clone https://github.com/aldefy/compose-skill.git /tmp/compose-skill

# 复制到全局 Antigravity skills 目录
mkdir -p ~/.gemini/antigravity/skills
cp -r /tmp/compose-skill/skills/compose-expert ~/.gemini/antigravity/skills/compose-expert
```

---

### Cursor

创建 `.cursor/rules/compose-skill.mdc`：

```markdown
---
description: Jetpack Compose expert guidance
globs: **/*.kt
---

所有 Compose 相关代码，遵循 `skills/compose-expert/SKILL.md` 中的指令，
并在建议模式前查阅 `skills/compose-expert/references/` 中的参考文件。
```

或将 `SKILL.md` 的内容粘贴到 **Settings > Rules for AI** 中。

---

### GitHub Copilot

添加到 `.github/copilot-instructions.md`：

```markdown
## Jetpack Compose
所有 Compose/Android UI 工作，遵循 `skills/compose-expert/SKILL.md` 中的 skill 指令。
查阅 `skills/compose-expert/references/` 中的参考文件获取模式、陷阱和源码级指导。
```

---

### Windsurf

在项目根目录创建 `.windsurf/rules/compose-skill.md`：

```markdown
所有 Jetpack Compose 任务，遵循 `skills/compose-expert/SKILL.md` 中的工作流，
并在回答前查阅 `skills/compose-expert/references/` 中的参考文件。
```

> **注意：** 旧版 `.windsurfrules` 文件也有效，但推荐使用 `.windsurf/rules/`。

---

### Amazon Q Developer

添加到 `.amazonq/rules/compose.md`：

```markdown
所有 Jetpack Compose 任务，遵循 `skills/compose-expert/SKILL.md` 中的工作流，
并在回答前查阅 `skills/compose-expert/references/` 中的参考文件。
```

---

### 其他 AI 编程工具

本质上就是 Markdown。将本仓库克隆到项目中（或添加为子模块），然后将工具的指令文件指向 `skills/compose-expert/SKILL.md` 即可。AI 会读取 `SKILL.md` 获取工作流，并根据需要从 `references/` 中提取内容。

## 快速示例

配置完成后，像平常一样与 AI 助手对话即可：

```
"我的 LazyColumn 滚动时很卡，帮我修一下"
```

会发生什么：
1. AI 读取 `SKILL.md` 获取工作流
2. 引入 `references/lists-scrolling.md` 和 `references/performance.md`
3. 检查你的代码是否缺少 `key {}`、列表项类型是否不稳定、item 块中是否有重型计算
4. 如有疑问，查证 `references/source-code/foundation-source.md`
5. 基于真实的 `LazyList` 实现给出修复方案

没有幻觉 API。没有猜测行为。

## 源码

源码来自 [`androidx/androidx`](https://github.com/androidx/androidx/tree/androidx-main/compose)，基于 Apache License 2.0。指导文档是对这些源码及官方文档的原创分析。

## 贡献

欢迎 PR，特别是：
- 额外的平台相关陷阱和解决方案
- 常见 UI 模式的新动画配方
- 来自真实应用的线上崩溃模式
- 基于新版 Compose 的修正
- 追踪新版本的自动更新工具

## 许可证

MIT —— 详见 [LICENSE](LICENSE)。

`androidx/androidx` 的源码基于 Apache License 2.0，版权属于 Android Open Source Project。
