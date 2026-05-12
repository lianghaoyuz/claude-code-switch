# 方案：为每个 Provider 分别指定细粒度模型字段

## 背景

当前，对于第三方 provider，`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL` 和 `CLAUDE_CODE_SUBAGENT_MODEL` 要么被硬编码，要么与 `ANTHROPIC_MODEL` 设为相同值。`CLAUDE_CODE_EFFORT_LEVEL` 则完全不会被导出。用户无法独立配置每个 Claude Code 模型层级对应的具体模型。

本次改造将在 `~/.ccm_config` 中新增按 provider 分类的细粒度配置变量，让用户可以单独设置每一个导出字段。

## 命名规范

```
{PROVIDER}_SONNET_MODEL    → ANTHROPIC_DEFAULT_SONNET_MODEL
{PROVIDER}_OPUS_MODEL      → ANTHROPIC_DEFAULT_OPUS_MODEL
{PROVIDER}_HAIKU_MODEL     → ANTHROPIC_DEFAULT_HAIKU_MODEL
{PROVIDER}_SUBAGENT_MODEL  → CLAUDE_CODE_SUBAGENT_MODEL
{PROVIDER}_EFFORT_LEVEL    → CLAUDE_CODE_EFFORT_LEVEL
```

适用 Provider：`DEEPSEEK`、`KIMI`、`KIMI_CN`、`GLM`、`QWEN`、`MINIMAX`、`SEED`、`STEPFUN`、`CLAUDE`

## 向后兼容策略

- **所有 provider（含 DeepSeek、Qwen）**：SONNET/OPUS/HAIKU 默认值统一等于 `ANTHROPIC_MODEL`（即主模型 `${PROVIDER}_MODEL`）
- **Claude 例外**：OPUS 回退链为 `CLAUDE_OPUS_MODEL` → `OPUS_MODEL` → `CLAUDE_MODEL`；HAIKU 回退链为 `CLAUDE_HAIKU_MODEL` → `HAIKU_MODEL` → `CLAUDE_MODEL`（保留现有 `OPUS_MODEL`/`HAIKU_MODEL` 变量兼容）
- **`_EFFORT_LEVEL`**：不设置就不导出该环境变量
- **`_SUBAGENT_MODEL`**：默认等于 `ANTHROPIC_MODEL`（保留当前行为）

## 涉及文件

- `/Users/lianghaoyu/Desktop/cctest1/claude-code-switch/ccm.sh`（唯一修改文件）

## 实施步骤

### 步骤 1：`clean_env()` 增加 `CLAUDE_CODE_EFFORT_LEVEL`（第 1493 行）
在已有 unset 块中添加 `unset CLAUDE_CODE_EFFORT_LEVEL`。

### 步骤 2：更新 `emit_env_exports()` 的 prelude（第 2071 行）
在 `prelude` 的 unset 字符串末尾追加 `CLAUDE_CODE_EFFORT_LEVEL`。

### 步骤 3：更新 `emit_env_exports()` 中所有 provider 分支（第 2077-2283 行）

**通用模式**：计算主模型后，用旧硬编码默认值解析各字段覆盖：

**deepseek**（第 2077 行）：
```bash
local ds_model="${DEEPSEEK_MODEL:-deepseek-chat}"
local ds_sonnet="${DEEPSEEK_SONNET_MODEL:-$ds_model}"
local ds_opus="${DEEPSEEK_OPUS_MODEL:-$ds_model}"
local ds_haiku="${DEEPSEEK_HAIKU_MODEL:-$ds_model}"
local ds_subagent="${DEEPSEEK_SUBAGENT_MODEL:-$ds_model}"
local ds_effort="${DEEPSEEK_EFFORT_LEVEL:-}"
```

**kimi**（第 2092 行）：global 用 `KIMI_*` 前缀，china 用 `KIMI_CN_*` 前缀。全部默认等于 `$kimi_model`。

**qwen**（第 2124 行）：
```bash
local qwen_sonnet="${QWEN_SONNET_MODEL:-$qwen_model}"
local qwen_opus="${QWEN_OPUS_MODEL:-$qwen_model}"
local qwen_haiku="${QWEN_HAIKU_MODEL:-$qwen_model}"
```

**glm**、**minimax**、**seed**、**stepfun**：全部默认等于主模型（旧行为）。

**claude**（第 2262 行）：
```bash
local claude_sonnet="${CLAUDE_SONNET_MODEL:-$claude_model}"
local claude_opus="${CLAUDE_OPUS_MODEL:-${OPUS_MODEL:-claude-opus-4-6}}"
local claude_haiku="${CLAUDE_HAIKU_MODEL:-${HAIKU_MODEL:-claude-haiku-4-5-20251001}}"
```

### 步骤 4：更新 `emit_openrouter_exports()`（约第 2051 行）
prelude 添加 `CLAUDE_CODE_EFFORT_LEVEL`。各 provider 同样增加细粒度字段解析。

### 步骤 5：更新 `user_write_settings()`（第 679 行）和 `project_write_settings()`（第 471 行）
从 `get_provider_config()` 取到配置后，建立 provider→前缀映射，解析各字段变量，在 JSON 模板中使用。条件性添加 `CLAUDE_CODE_EFFORT_LEVEL`。

### 步骤 6：更新 `project_write_glm_settings()`（第 400 行）
同理，解析 GLM 的各字段覆盖，写入 heredoc JSON 模板。

### 步骤 7：更新 `switch_to_*` 旧函数（第 1509-1758 行）
虽 `main()` 已路由到 `emit_env_exports()`，这些函数仍保留供外部调用。更新每个函数以解析各字段覆盖。

### 步骤 8：更新 `load_config()` 中的配置模板（约第 141-188 行）
添加所有新可选变量的注释区域。

### 步骤 9：更新 `show_status()`（第 1391 行）
显示 `CLAUDE_CODE_EFFORT_LEVEL`（如已设置）。

## 验证方案

1. **语法检查**: `bash -n ccm.sh`
2. **默认行为**: 不设置任何新变量，运行 `ccm deepseek` — 验证 SONNET/OPUS/HAIKU 均等于 `DEEPSEEK_MODEL`（默认 `deepseek-chat`）
3. **新变量生效**: 在配置中设 `DEEPSEEK_HAIKU_MODEL=deepseek-v4-flash`，运行 `ccm deepseek` — 验证 HAIKU 用了覆盖值，而 SONNET/OPUS 仍等于主模型
4. **EFFORT_LEVEL**: 设 `DEEPSEEK_EFFORT_LEVEL=3` — 验证导出了 `CLAUDE_CODE_EFFORT_LEVEL`；移除 — 验证不导出
5. **Claude 兼容**: 现有 `OPUS_MODEL`/`HAIKU_MODEL` 在 `ccm claude` 时仍生效
6. **全 provider 覆盖**: 每个 provider 用 `ccm status` 验证模型解析正确
7. **用户/项目级设置**: `ccm user deepseek` 和 `ccm project deepseek` 写入正确的各字段值
