# Prompt for Codex / Claude

> 将以下内容完整粘贴给 Codex 或 Claude，它会自动完成所有配置。
> 开始前请准备好你的 **MiMo API Key**。

---

## Prompt

```
你的任务是为我配置 Claude Code + MiMo Token-Plan 的图片识别方案。

背景：mimo-v2.5-pro 不支持图片输入，需要通过 CCR（Claude Code Router）做智能路由，配合 mimo-image-recognition MCP Server 解决图片识别问题。

请按以下步骤执行，每一步完成后告诉我结果，遇到错误立即停止并告诉我原因。

## 前置检查

1. 检测当前操作系统（macOS / Linux / Windows）和 Shell（zsh / bash / powershell）
2. 检查 Node.js 是否已安装（`node -v`），未安装则提示我先安装
3. 检查 npm 是否可用（`npm -v`）
4. 向我询问 MiMo API Key（格式类似 sk-xxxxx）

## Step 1：安装 CCR

```bash
npm install -g @musistudio/claude-code-router
```

安装后验证：`ccr -v`

## Step 2：创建 CCR 配置

创建 `~/.claude-code-router/config.json`，内容如下（注意替换 CUSTOM_ROUTER_PATH 中的用户名）：

```json
{
  "LOG": false,
  "LOG_LEVEL": "info",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": 600000,
  "NON_INTERACTIVE_MODE": false,
  "Providers": [
    {
      "name": "mimo",
      "api_base_url": "https://token-plan-cn.xiaomimimo.com/anthropic/v1/messages",
      "api_key": "$ANTHROPIC_API_KEY",
      "models": [
        "mimo-v2.5-pro",
        "mimo-v2.5"
      ],
      "transformer": {
        "use": [
          "Anthropic"
        ]
      }
    }
  ],
  "Router": {
    "default": "mimo,mimo-v2.5-pro",
    "background": "mimo,mimo-v2.5-pro",
    "think": "mimo,mimo-v2.5-pro",
    "longContext": "mimo,mimo-v2.5-pro",
    "longContextThreshold": 1000000,
    "webSearch": "mimo,mimo-v2.5-pro",
    "image": "mimo,mimo-v2.5"
  },
  "CUSTOM_ROUTER_PATH": "<HOME>/.claude-code-router/custom-router.js"
}
```

将 `<HOME>` 替换为 `~` 的实际绝对路径（如 `/Users/xxx` 或 `/home/xxx`）。

## Step 3：存储 API Key

根据操作系统选择：

- **macOS**：`security add-generic-password -a "$USER" -s "mimo_api_key" -w "<API_KEY>" -U`
- **Linux**：先检查 `secret-tool` 是否可用，不可用则提示 `sudo apt install libsecret-tools`，然后 `echo "<API_KEY>" | secret-tool store --label="mimo_api_key" service mimo_api_key username "$USER"`
- **Windows (PowerShell)**：`[Environment]::SetEnvironmentVariable("MIMO_API_KEY", "<API_KEY>", "User")`

## Step 4：创建 MCP 配置

创建或合并 `~/.claude/.mcp.json`：

```json
{
  "mcpServers": {
    "mimo-image-recognition": {
      "command": "uvx",
      "args": ["--refresh", "mimo-image-recognition-mcp"],
      "env": {
        "MIMO_API_KEY": "<API_KEY>",
        "MIMO_API_BASE": "https://token-plan-cn.xiaomimimo.com/v1",
        "MIMO_MODEL": "mimo-v2.5"
      }
    }
  }
}
```

注意：如果该文件已存在其他 MCP 配置，只添加 `mimo-image-recognition` 部分，不要覆盖已有内容。

## Step 5：创建环境变量文件

创建 `~/.config/mimo/env`（目录不存在则先创建），内容如下：

```bash
# MiMo API settings for Claude Code.
export ANTHROPIC_BASE_URL="https://token-plan-cn.xiaomimimo.com/anthropic"
export ANTHROPIC_MODEL="opus"

export ANTHROPIC_DEFAULT_OPUS_MODEL="mimo-v2.5-pro"
export ANTHROPIC_DEFAULT_OPUS_MODEL_NAME="mimo-v2.5-pro"
export ANTHROPIC_DEFAULT_OPUS_MODEL_DESCRIPTION="MiMo Token-Plan Pro"
export ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES="effort,xhigh_effort,thinking,adaptive_thinking,interleaved_thinking"

export ANTHROPIC_DEFAULT_SONNET_MODEL="mimo-v2.5"
export ANTHROPIC_DEFAULT_SONNET_MODEL_NAME="mimo-v2.5"
export ANTHROPIC_DEFAULT_SONNET_MODEL_DESCRIPTION="MiMo Token-Plan"
export ANTHROPIC_DEFAULT_SONNET_MODEL_SUPPORTED_CAPABILITIES="effort,xhigh_effort,thinking,adaptive_thinking,interleaved_thinking"

export ANTHROPIC_DEFAULT_HAIKU_MODEL="mimo-v2.5-asr"
export ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME="mimo-v2.5-asr"
export ANTHROPIC_DEFAULT_HAIKU_MODEL_DESCRIPTION="MiMo Token-Plan ASR"

export ANTHROPIC_CUSTOM_MODEL_OPTION="mimo-v2.5-tts-voiceclone"
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="mimo-v2.5-tts-voiceclone"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="MiMo Token-Plan TTS Voice Clone"

# 跨平台 API Key 读取
_mimo_api_key=""

if [ -z "$_mimo_api_key" ] && [ "$(uname -s)" = "Darwin" ] && command -v security >/dev/null 2>&1; then
  _mimo_api_key="$(security find-generic-password -a "$USER" -s "mimo_api_key" -w 2>/dev/null || true)"
fi

if [ -z "$_mimo_api_key" ] && [ "$(uname -s)" = "Linux" ] && command -v secret-tool >/dev/null 2>&1; then
  _mimo_api_key="$(secret-tool lookup service mimo_api_key username "$USER" 2>/dev/null || true)"
fi

if [ -z "$_mimo_api_key" ] && [[ "$(uname -s)" == MINGW* || "$(uname -s)" == MSYS* || "$(uname -s)" == CYGWIN* ]]; then
  _mimo_api_key="${MIMO_API_KEY:-}"
fi

_mimo_api_key="${_mimo_api_key:-${MIMO_API_KEY:-}}"

if [ -n "$_mimo_api_key" ]; then
  export ANTHROPIC_API_KEY="$_mimo_api_key"
fi
unset _mimo_api_key
```

## Step 6：配置 Shell 自动启动

根据 Shell 类型，在对应 rc 文件末尾追加（不要覆盖原有内容）：

**zsh (`~/.zshrc`) 或 bash (`~/.bashrc`)：**

```bash
# ============================================
# MiMo API 环境变量（按平台读取密钥）
# ============================================
[ -f "$HOME/.config/mimo/env" ] && source "$HOME/.config/mimo/env"

# ============================================
# Claude Code 模型别名与包装函数
# ============================================
[ -f "$HOME/.config/mimo/claude.zsh" ] && [ -n "$ZSH_VERSION" ] && source "$HOME/.config/mimo/claude.zsh"
[ -f "$HOME/.config/mimo/claude.bash" ] && [ -n "$BASH_VERSION" ] && source "$HOME/.config/mimo/claude.bash"

# ============================================
# CCR 自动启动：确保 Claude Code 通过 CCR 代理
# ============================================
if command -v ccr >/dev/null 2>&1; then
  ccr status 2>/dev/null | grep -q "Status: Running" || ccr restart >/dev/null 2>&1
  eval "$(ccr activate)"
fi
```

**PowerShell (`$PROFILE`)：**

```powershell
# MiMo API 环境变量
$env:ANTHROPIC_BASE_URL = "https://token-plan-cn.xiaomimimo.com/anthropic"
$env:ANTHROPIC_API_KEY = [Environment]::GetEnvironmentVariable("MIMO_API_KEY", "User")
$env:ANTHROPIC_MODEL = "opus"

$env:ANTHROPIC_DEFAULT_OPUS_MODEL = "mimo-v2.5-pro"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL_NAME = "mimo-v2.5-pro"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL = "mimo-v2.5"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL_NAME = "mimo-v2.5"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL = "mimo-v2.5-asr"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME = "mimo-v2.5-asr"

# CCR 自动启动
if (Get-Command ccr -ErrorAction SilentlyContinue) {
    $status = ccr status 2>&1
    if ($status -notmatch "Status: Running") { ccr restart 2>$null }
    Invoke-Expression (ccr activate)
}
```

追加前先检查 rc 文件末尾是否已有相同内容，避免重复追加。

## Step 7：配置 Claude Code 权限

检查 `~/.claude/settings.local.json` 是否存在：
- 不存在：创建完整文件
- 已存在：只在 `permissions.allow` 数组中追加，不覆盖其他配置

```json
{
  "permissions": {
    "allow": [
      "mcp__mimo-image-recognition__understand_image"
    ]
  }
}
```

## Step 8：启动 CCR 并验证

1. 运行 `ccr restart`
2. 运行 `ccr status`，确认 Status 为 Running
3. 运行 `echo $ANTHROPIC_BASE_URL`，确认输出 `http://127.0.0.1:3456`（CCR 代理地址）

## 完成

全部完成后，告诉我：
- 哪些步骤成功了
- 哪些步骤有问题
- 最终提示我：打开新终端，运行 `claude`，粘贴一张图片测试识别
```
