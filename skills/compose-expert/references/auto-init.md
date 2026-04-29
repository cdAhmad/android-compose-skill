# 自动初始化：Compose 项目检测

在 `session_start` 时激活。检测当前项目是否使用 Compose，
并以简短提示静默激活 skill。

---

## 检测门控

按顺序运行，首次匹配即停止。

### 步骤 1 — Gradle 扫描

在工作目录或上级目录中查找 `build.gradle.kts`、`build.gradle` 或 `libs.versions.toml`。
检查文件内容是否包含以下任一关键词：

- `compose`
- `androidx.compose`

```bash
# 检查工作目录及上级目录
for dir in . ..; do
  for file in build.gradle.kts build.gradle libs.versions.toml; do
    if [ -f "$dir/$file" ]; then
      grep -qi "compose\|androidx\.compose" "$dir/$file" && echo "DETECTED" && break 2
    fi
  done
done
```

### 步骤 2 — 源码扫描兜底

如果未找到 Gradle 文件或 Gradle 中无 Compose 引用，则扫描 Kotlin 源文件。

```bash
# 查找最多 10 个 .kt 文件（排除 build 目录），检查是否包含 @Composable
find . -name "*.kt" -not -path "*/build/*" -print -quit 2>/dev/null | head -10 | \
  xargs grep -l "@Composable" 2>/dev/null | head -1
```

如果任一文件包含 `@Composable`，检测成功。

---

## 检测到后

输出一行提示：

```
Compose project detected — compose-expert skill active.
```

然后正常继续 —— 等待用户请求并遵循 SKILL.md 中的标准工作流。

## 未检测到

不执行任何操作。如果用户在会话后期通过关键词显式触发
（例如提及 `@Composable`、`LazyColumn`、`NavHost` 等），
skill 仍然可用。
