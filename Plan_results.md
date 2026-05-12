# CCM 细粒度模型字段配置 — 实施结果

## 目标

在 `~/.ccm_config` 中为每个 provider 新增细粒度配置变量，允许用户分别指定以下导出字段：

- `ANTHROPIC_DEFAULT_SONNET_MODEL`
- `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `ANTHROPIC_DEFAULT_HAIKU_MODEL`
- `CLAUDE_CODE_SUBAGENT_MODEL`
- `CLAUDE_CODE_EFFORT_LEVEL`

## 修改文件

- `ccm.sh`（唯一修改文件）

## 新增配置变量

每个 provider 支持以下可选变量（不设置时回退到主模型 `${PROVIDER}_MODEL`）：

```
{PROVIDER}_SONNET_MODEL    → ANTHROPIC_DEFAULT_SONNET_MODEL
{PROVIDER}_OPUS_MODEL      → ANTHROPIC_DEFAULT_OPUS_MODEL
{PROVIDER}_HAIKU_MODEL     → ANTHROPIC_DEFAULT_HAIKU_MODEL
{PROVIDER}_SUBAGENT_MODEL  → CLAUDE_CODE_SUBAGENT_MODEL
{PROVIDER}_EFFORT_LEVEL    → CLAUDE_CODE_EFFORT_LEVEL
```

适用 provider：`DEEPSEEK`、`KIMI`、`KIMI_CN`（Kimi China）、`GLM`、`QWEN`、`MINIMAX`、`SEED`、`STEPFUN`、`CLAUDE`

## 详细修改清单

### 1. 环境清理和 prelude 字符串

| 函数 | 行号 | 修改内容 |
|---|---|---|
| `clean_env()` | ~1577 | 新增 `unset CLAUDE_CODE_EFFORT_LEVEL` |
| `emit_env_exports()` prelude | ~2072 | 追加 `CLAUDE_CODE_EFFORT_LEVEL` 到 unset 列表 |
| `emit_openrouter_exports()` prelude | ~2052 | 追加 `CLAUDE_CODE_EFFORT_LEVEL` 到 unset 列表 |

### 2. `emit_env_exports()` — 各 provider 分支

所有 8 个 provider 均按相同模式修改：

```bash
# 新增局部变量
local ${prefix}_sonnet="${${PREFIX}_SONNET_MODEL:-$main_model}"
local ${prefix}_opus="${${PREFIX}_OPUS_MODEL:-$main_model}"
local ${prefix}_haiku="${${PREFIX}_HAIKU_MODEL:-$main_model}"
local ${prefix}_subagent="${${PREFIX}_SUBAGENT_MODEL:-$main_model}"
local ${prefix}_effort="${${PREFIX}_EFFORT_LEVEL:-}"

# 使用覆盖值导出
emit_default_models "$sonnet" "$opus" "$haiku"
emit_subagent_model "$subagent"
[[ -n "$effort" ]] && echo "export CLAUDE_CODE_EFFORT_LEVEL='${effort}'"
```

| 分支 | 前缀 | 特殊情况 |
|---|---|---|
| deepseek/ds | `DEEPSEEK_` | — |
| kimi global | `KIMI_` | 按 region 分支处理 |
| kimi china | `KIMI_CN_` | 独立前缀 |
| qwen | `QWEN_` | — |
| glm | `GLM_` | — |
| minimax/mm | `MINIMAX_` | — |
| seed/doubao | `SEED_` | 覆盖在 variant 分支后生效 |
| stepfun | `STEPFUN_` | — |
| claude/sonnet/s | `CLAUDE_` | OPUS 回退链: `CLAUDE_OPUS_MODEL` → `OPUS_MODEL` → 主模型 |
| | | HAIKU 回退链: `CLAUDE_HAIKU_MODEL` → `HAIKU_MODEL` → 主模型 |

### 3. `emit_openrouter_exports()` — OpenRouter 统一覆盖

在 case 分支后新增 provider→prefix 映射表和统一解析逻辑，避免逐个修改每个 case：

```bash
# 映射 provider → 前缀
case "$provider" in
    "claude"|"anthropic")  → CLAUDE
    "kimi")                → KIMI
    "deepseek"|"ds")       → DEEPSEEK
    ...
esac

# 用间接变量扩展读取配置，回退到 OpenRouter 硬编码默认值
local resolved_sonnet="${!sonnet_var:-$default_sonnet}"
```

### 4. `user_write_settings()` / `project_write_settings()`

两个函数均新增 provider→prefix 映射（与上述相同），在写入 `settings.json` / `settings.local.json` 时使用解析后的各字段覆盖值。

- Python 路径（user_write_settings）：`existing['env']` dict 改用 `$field_sonnet` 等变量，条件性添加 `CLAUDE_CODE_EFFORT_LEVEL`
- Heredoc 回退路径：JSON 模板改用各字段变量，条件性追加 EFFORT_LEVEL 行

### 5. `project_write_glm_settings()`

新增 GLM 的各字段覆盖解析，heredoc JSON 模板改用独立变量。

### 6. `switch_to_*` 遗留函数

全部 8 个函数均按相同模式更新：

| 函数 | 前缀 |
|---|---|
| `switch_to_deepseek()` | `DEEPSEEK_` |
| `switch_to_claude()` | `CLAUDE_`（含 OPUS_MODEL/HAIKU_MODEL 回退） |
| `switch_to_glm()` | `GLM_` |
| `switch_to_kimi()` | `KIMI_` |
| `switch_to_kimi_cn()` | `KIMI_CN_` |
| `switch_to_minimax()` | `MINIMAX_` |
| `switch_to_qwen()` | `QWEN_` |
| `switch_to_seed()` | `SEED_` |

### 7. `show_status()` — 状态显示

新增一行显示 `EFFORT_LEVEL`：
```
echo "   EFFORT_LEVEL: ${CLAUDE_CODE_EFFORT_LEVEL:-'(not set)'}"
```

### 8. 配置模板（两处）

在已有的模型覆盖段落后新增注释提示：
```
# —— 可选：细粒度模型字段（不设置则使用主模型）——
# 可选字段: _SONNET_MODEL, _OPUS_MODEL, _HAIKU_MODEL, _SUBAGENT_MODEL, _EFFORT_LEVEL
# 用法: DEEPSEEK_HAIKU_MODEL=deepseek/deepseek-v3.2
```

## 配置使用示例

在 `~/.ccm_config` 中添加：

```bash
# 为 DeepSeek 设置不同层级的模型
DEEPSEEK_MODEL=deepseek-chat
DEEPSEEK_SONNET_MODEL=deepseek/deepseek-v3.2
DEEPSEEK_HAIKU_MODEL=deepseek/deepseek-v3.2:free

# 为 Kimi China 设置 effort level
KIMI_CN_EFFORT_LEVEL=3

# 为 Qwen 设置独立子代理模型
QWEN_SUBAGENT_MODEL=qwen3-coder-plus
```

## 向后兼容说明

- **所有新变量均为可选**：不设置时行为完全不变
- **默认值策略**：所有 provider（含 DeepSeek、Qwen）的 SONNET/OPUS/HAIKU 默认回退到 `${PROVIDER}_MODEL`
- **Claude 保留**：现有 `OPUS_MODEL` 和 `HAIKU_MODEL` 变量继续生效，作为 `CLAUDE_OPUS_MODEL` / `CLAUDE_HAIKU_MODEL` 的二级回退
- **EFFORT_LEVEL**：不设置则不导出，Claude Code 使用其内置默认值

## 验证记录

- [x] `bash -n ccm.sh` 语法检查通过
- [x] 默认行为不变（SONNET/OPUS/HAIKU/SUBAGENT 全部等于主模型）
- [x] 自定义覆盖生效（`DEEPSEEK_HAIKU_MODEL`、`DEEPSEEK_EFFORT_LEVEL`）
- [x] Kimi 双区域独立前缀（`KIMI_*` vs `KIMI_CN_*`）
- [x] Claude `OPUS_MODEL`/`HAIKU_MODEL` 向后兼容
- [x] `ccm status` 正确显示 `EFFORT_LEVEL` 字段
