# <img src="https://img.shields.io/badge/Claude--Code--Profile--Manager-Windows-00b0f0?style=flat-square&logo=microsoft&logoColor=white" alt="Platform: Windows/PowerShell"> Claude-Code-Profile-Manager (Windows Edition)

<div align="center">

**Platform** | **Version** | **License** | **Author**
:---:|:---:|:---:|:---:
Windows / PowerShell | MIT | Cloud927

让 Windows 终端秒变多模型 AI 启动器

</div>

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Usage Guide](#usage-guide)
  - [Add Model](#-add-model)
  - [List Models](#-list-models)
  - [Remove Model](#-remove-model)
- [Troubleshooting](#troubleshooting)
  - [Auth Conflict (Token 冲突)](#auth-conflict-token-冲突)
  - [清理旧配置](#清理旧配置)
- [Model Configuration Cheat Sheet](#model-configuration-cheat-sheet)

---

## 🎯 Project Overview

在 Windows 上直接使用 `claude-code` 切换 DeepSeek、Kimi 等模型极其繁琐 —— Windows 环境变量的临时配置远比 Linux 复杂。**CCPM** 专为 PowerShell 设计的自动化脚本，让多模型管理变得像呼吸一样自然。

---

## ✨ Key Features

| Feature | Description |
|---------|-------------|
| 🛡️ **环境隔离** | 自动"挂载"和"卸载" API Key，不污染系统环境变量，安全可控 |
| ⚡️ **秒级切换** | `cdsc` 启动 DeepSeek，`ckm` 启动 Kimi，命令即开即用 |
| 🔄 **自动装配** | `Add-Model` 向导式配置，交互式填写 API Key 和模型参数 |
| 🚫 **防冲突机制** | 内置 Token 冲突检测与自动清理，避免配置相互覆盖 |
| 💾 **持久化存储** | 配置文件独立存放于 `~/.claude_profiles/`，便于备份和迁移 |

---

## 📦 Prerequisites

| Dependency | Version | Required |
|------------|---------|----------|
| **Node.js** | >= 18.x | ✅ |
| **PowerShell** | >= 5.0 | ✅ |
| **Claude Code CLI** | Latest | ✅ |

### 检查 Node.js 是否已安装

```powershell
node -v
```

> 如果显示找不到命令，请前往 [Node.js 官网](https://nodejs.org/) 下载 **LTS 版本**并安装，安装后重启终端。

### 安装 Claude Code CLI

```powershell
npm install -g @anthropic-ai/claude-code
```

验证安装：

```powershell
claude --version
```

---

## 🚀 Quick Start

### Step 1: 打开 PowerShell 配置文件

```powershell
if (!(Test-Path $PROFILE)) { New-Item -Type File -Path $PROFILE -Force }
notepad $PROFILE
```

### Step 2: 注入核心代码

将以下代码**完整复制**到打开的 `$PROFILE` 文件中，保存并关闭：

```powershell
# --- Claude Code Profile Manager (CCPM) ---

# 初始化配置目录
$ConfigDir = "$HOME\.claude_profiles"
if (!(Test-Path $ConfigDir)) { New-Item -ItemType Directory -Path $ConfigDir -Force | Out-Null }

# 1. 核心启动器
function Global:Run-Claude ($ProfileName) {
    $Path = "$HOME\.claude_profiles\$ProfileName.conf"
    if (Test-Path $Path) {
        $Config = Get-Content $Path -Raw -Encoding UTF8 | ConvertFrom-StringData

        # 保存当前环境状态
        $OldKey = $env:ANTHROPIC_API_KEY
        $OldUrl = $env:ANTHROPIC_BASE_URL
        $OldModel = $env:ANTHROPIC_MODEL
        $OldToken = $env:ANTHROPIC_AUTH_TOKEN

        # 强制清除冲突 Token
        $env:ANTHROPIC_AUTH_TOKEN = $null

        # 注入新配置
        $env:ANTHROPIC_API_KEY = $Config.KEY
        if ($Config.URL) { $env:ANTHROPIC_BASE_URL = $Config.URL }
        if ($Config.MODEL) { $env:ANTHROPIC_MODEL = $Config.MODEL }

        Write-Host ("🚀 Launching: " + $Config.DISPLAY_NAME) -ForegroundColor Cyan

        # 启动 Claude
        claude

        # 恢复环境状态
        $env:ANTHROPIC_API_KEY = $OldKey
        $env:ANTHROPIC_BASE_URL = $OldUrl
        $env:ANTHROPIC_MODEL = $OldModel
        $env:ANTHROPIC_AUTH_TOKEN = $OldToken
    } else {
        Write-Host ("❌ Config not found: " + $ProfileName) -ForegroundColor Red
    }
}

# 2. 添加模型向导
function Global:Add-Model {
    Write-Host "--- Add New Model ---" -ForegroundColor Yellow

    $ShortName = Read-Host "1. Short Name (e.g. dsc)"
    if (!$ShortName) { return }
    if (Test-Path "$HOME\.claude_profiles\$ShortName.conf") {
        Write-Host "Error: Profile already exists" -ForegroundColor Red
        return
    }

    $DisplayName = Read-Host "2. Display Name (e.g. DeepSeek)"
    if (!$DisplayName) { $DisplayName = $ShortName }

    $ApiKey = Read-Host "3. API Key"
    $BaseUrl = Read-Host "4. Base URL (Enter to skip)"
    $ModelId = Read-Host "5. Model ID"

    $Content = "DISPLAY_NAME=$DisplayName`nKEY=$ApiKey"
    if ($BaseUrl) { $Content += "`nURL=$BaseUrl" }
    if ($ModelId) { $Content += "`nMODEL=$ModelId" }

    Set-Content -Path "$HOME\.claude_profiles\$ShortName.conf" -Value $Content -Encoding UTF8

    # 动态创建快捷命令
    Invoke-Expression "function Global:c$ShortName { Run-Claude '$ShortName' }"

    Write-Host ("✅ Success! Type c$ShortName to launch.") -ForegroundColor Green
}

# 3. 移除模型
function Global:Remove-Model {
    $ShortName = Read-Host "Delete which model? (short name)"
    $Path = "$HOME\.claude_profiles\$ShortName.conf"
    if (Test-Path $Path) {
        Remove-Item $Path
        Remove-Item "function:c$ShortName" -ErrorAction SilentlyContinue
        Write-Host "✅ Deleted." -ForegroundColor Green
    } else {
        Write-Host "❌ Not found." -ForegroundColor Red
    }
}

# 4. 列出所有模型
function Global:Models {
    Write-Host "----------------------------------------"
    Write-Host "🤖 Claude Code Profiles"
    Write-Host "----------------------------------------"
    Write-Host "Command          Model Name"
    Write-Host "----------------------------------------"

    Get-ChildItem "$HOME\.claude_profiles\*.conf" | ForEach-Object {
        $Name = $_.BaseName
        $Conf = Get-Content $_.FullName -Raw -Encoding UTF8 | ConvertFrom-StringData
        $Cmd = "c$Name"
        $PaddedCmd = $Cmd.PadRight(16)
        $DName = $Conf.DISPLAY_NAME
        Write-Host ($PaddedCmd + " " + $DName)
    }
    Write-Host "----------------------------------------"
    Write-Host "Add:    Add-Model    |    Remove: Remove-Model"
}

# 5. 启动时自动加载所有配置
Get-ChildItem "$HOME\.claude_profiles\*.conf" | ForEach-Object {
    $Name = $_.BaseName
    if (-not (Get-Command "c$Name" -ErrorAction SilentlyContinue)) {
        Invoke-Expression "function Global:c$Name { Run-Claude '$Name' }"
    }
}
```

### Step 3: 使配置生效

```powershell
. $PROFILE
```

> ⚠️ 如果遇到权限错误，请先执行：`Set-ExecutionPolicy RemoteSigned`（以管理员身份运行）

---

## 📖 Usage Guide

### ➕ Add Model

```powershell
Add-Model
```

按提示填写：

| Step | Input | Example |
|------|-------|---------|
| 1 | Short Name | `dsc` |
| 2 | Display Name | `DeepSeek-Chat` |
| 3 | API Key | `sk-xxxxxxxxx` |
| 4 | Base URL | `https://api.deepseek.com/v1` |
| 5 | Model ID | `deepseek-chat` |

### 📋 List Models

```powershell
Models
```

输出示例：

```
----------------------------------------
🤖 Claude Code Profiles
----------------------------------------
Command          Model Name
----------------------------------------
cdsc             DeepSeek-Chat
ckm              Kimi-K2.5
----------------------------------------
Add:    Add-Model    |    Remove: Remove-Model
```

### ➖ Remove Model

```powershell
Remove-Model
```

输入要删除的模型简称即可。

---

## 🔧 Troubleshooting

### Auth Conflict (Token 冲突)

**症状**：启动时出现黄色警告 `Auth conflict`

**原因**：脚本正在覆盖官方配置以使用你自定义的 API Key

**解决方案**：

```powershell
# 清理可能残留的旧 Token
$env:ANTHROPIC_AUTH_TOKEN = $null

# 重新加载配置
. $PROFILE
```

> ✅ 这是正常行为，脚本已内置冲突处理机制，无需额外操作。

### 清理旧配置

如果遇到配置混乱或想重置所有设置：

```powershell
# 1. 删除所有配置文件
Remove-Item "$HOME\.claude_profiles\*.conf" -Force

# 2. 重新打开配置文件
notepad $PROFILE

# 3. 删除文件中的 CCPM 代码块（从 "# --- Claude Code Profile Manager ---" 到文件末尾）

# 4. 保存后重新加载
. $PROFILE
```

> 💡 **建议**：定期备份 `$HOME\.claude_profiles\` 目录，换电脑可直接恢复。

---

## 📝 Model Configuration Cheat Sheet

| Provider | Base URL | Model ID |
|----------|----------|----------|
| **DeepSeek** | `https://api.deepseek.com/v1` | `deepseek-chat` / `deepseek-reasoner` |
| **MiniMax** | `https://api.minimaxi.com/anthropic` | `MiniMax-M2.1` |
| **Kimi** | `https://api.moonshot.cn/anthropic` | `kimi-k2.5` |
| **Claude (Official)** | *(skip)* | `claude-3-5-sonnet-latest` |

---

<div align="center">

**Happy Coding on Windows!** 🚀

</div>
