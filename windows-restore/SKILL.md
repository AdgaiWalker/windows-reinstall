---
name: windows-restore
description: >
  Windows 全盘重装后的环境恢复向导。读取 machine-profile.json，按 discovered_sources 列表
  逐个恢复数据源。自动处理路径去重、盘符变化、文件夹重定向。
  当用户提到"重装系统后恢复"、"新系统环境配置"、"还原备份"、"从备份恢复"、
  "刚装完系统"、"帮我恢复环境"、"按照备份恢复"时触发。
  即使用户没明确说"恢复"，只要上下文表明是全新 Windows 需要还原环境，就应该触发。
  支持任何人使用：不需要懂技术，能看懂中文就行。
---

# Windows 环境恢复向导

重装完系统了？别慌，你的数据都在备份盘上。我来帮你找回来。

## 核心原则

**人类主权**：人类决定恢复什么、装到哪里。我不自行安装软件或修改系统配置。

**做减法**：只恢复需要的，不装暂时不用的工具。

**按数据源恢复**：备份时按 discovered_sources 记录了每个数据源，恢复时也按这个列表逐个处理。

**去重感知**：备份时如果桌面和下载路径相同做了合并，恢复时不会重复恢复。

## AI 能力边界

AI 能做的：复制文件、安装命令行工具、恢复配置、验证完整性。
AI 做不了的：安装需要点击的软件（浏览器、输入法、微信等）。

```
人类自己装日常应用（浏览器、微信、输入法...）
        ↓
AI 按 discovered_sources 逐个恢复数据
        ↓
AI 验证完整性（检查每个数据源是否恢复成功）
```

---

## 恢复前置：找到备份 + 读取 profile

### 找到 profile

外接盘插上后，盘符可能变了。

```powershell
$profilePath = $null
$drives = Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Root -match "^[D-Z]:" }
foreach ($d in $drives) {
    $found = Get-ChildItem "$($d.Root)" -Recurse -Depth 3 -Filter "machine-profile.json" -ErrorAction SilentlyContinue
    if ($found) { $profilePath = $found[0]; break }
}

if ($profilePath) { Write-Output "找到备份: $($profilePath.FullName)" }
else { Write-Output "没找到备份文件。请确认外接盘已插入，或告诉我备份在哪个路径。" }
```

### 读取 profile（带容错）

```powershell
try {
    $profile = Get-Content $profilePath.FullName -Raw -Encoding UTF8 | ConvertFrom-Json
} catch {
    Write-Output "备份文件损坏了。但别担心，备份目录下还有 personal/、dev/ 等文件夹。"
    Write-Output "我们可以手动恢复——你告诉我哪些目录有，我来帮你复制。"
    return
}
```

### 判断重装范围

读取 `profile.affected_drives` 和 `profile.disks`：

```powershell
$currentDrives = Get-Volume | Where-Object { $_.DriveLetter } | ForEach-Object { "$($_.DriveLetter):" }
$affectedDrives = $profile.affected_drives
```

向用户确认：
```
备份记录显示重装影响了：C:
当前系统检测到的盘：C: D: E: F:

请确认：其他盘的数据还在吗？（D/E/F 盘有没有被格式化？）
```

---

## 第一层：按数据源逐个恢复

核心逻辑：遍历 `profile.discovered_sources`，跳过 `backup: false` 的，逐个恢复。

### 恢复策略

对每个数据源，恢复策略取决于：
1. 原始路径的盘还在不在
2. 该盘是否被格式化了
3. 数据源类型（特殊文件夹需要重定向，应用数据不需要）

```
对每个数据源：
  原始盘还在？
    ├── 是 → 被格式化了？
    │         ├── 是 → 恢复到原位
    │         └── 否 → 跳过（数据还在）
    └── 否 → 降级到新系统默认位置，通知用户
```

### 通用恢复函数

```powershell
function Restore-DataSource {
    param(
        [PSObject]$Source,
        [string]$BackupRoot,
        [string[]]$FormattedDrives
    )

    # 跳过被合并的数据源
    if ($Source.backup -eq $false) {
        Write-Output "⏭ $($Source.name) — $($Source.note)"
        return @{ status="skipped"; reason=$Source.note }
    }

    # 找到备份目录
    $backupPath = Join-Path $BackupRoot $Source.backup_target
    if (-not (Test-Path $backupPath)) {
        Write-Output "⚠ $($Source.name) — 备份目录不存在"
        return @{ status="missing" }
    }

    # 确定恢复目标路径
    $originalPath = $Source.path
    $driveLetter = $originalPath.Substring(0,1)
    $driveExists = Test-Path "${driveLetter}:"
    $wasFormatted = $FormattedDrives -contains "${driveLetter}:"

    $target = $null
    $restoreNote = ""

    if (-not $driveExists) {
        # 原始盘不在了，降级到默认位置
        $target = Get-DefaultPath -SourceId $Source.id
        $restoreNote = "原始路径 $originalPath 的盘不在了，恢复到 $target"
    } elseif ($wasFormatted) {
        # 被格式化了，恢复到原位
        $target = $originalPath
        $restoreNote = "恢复到原位 $target"
    } else {
        # 盘还在且没格式化，跳过
        Write-Output "⏭ $($Source.name) — 盘 ${driveLetter}: 未被格式化，数据应该还在"
        return @{ status="skipped"; reason="盘未被格式化" }
    }

    # 执行恢复
    Write-Output "📦 $($Source.name) — $restoreNote"

    # 确保目标目录存在
    if (-not (Test-Path $target)) {
        New-Item -ItemType Directory -Path $target -Force | Out-Null
    }

    robocopy "$backupPath" "$target" /E /MT:8 /R:1 /W:1
    if ($LASTEXITCODE -ge 8) {
        Write-Output "❌ $($Source.name) — robocopy 失败 (exit code $LASTEXITCODE)"
        return @{ status="failed" }
    }

    return @{ status="ok"; target=$target }
}

function Get-DefaultPath {
    param([string]$SourceId)
    switch ($SourceId) {
        "desktop"   { [Environment]::GetFolderPath("Desktop") }
        "documents" { [Environment]::GetFolderPath("MyDocuments") }
        "downloads" {
            $shell = New-Object -ComObject Shell.Application
            $shell.Namespace('shell:Downloads').Self.Path
        }
        "pictures"  { [Environment]::GetFolderPath("MyPictures") }
        default     { "$env:USERPROFILE\restored\$SourceId" }
    }
}
```

### 执行恢复

```powershell
# 问用户确认哪些盘被格式化了
# (基于 affected_drives 和当前盘状态推断，让用户确认)

$formattedDrives = @("C:")  # 从用户确认中获取

foreach ($source in $profile.discovered_sources) {
    $result = Restore-DataSource -Source $source -BackupRoot $backupRoot -FormattedDrives $formattedDrives
    # 记录结果用于验证
}
```

### 微信恢复的特殊处理

微信数据恢复有特殊要求：**必须先登录微信一次再退出**。

```powershell
if ($source.id -eq "wechat" -and $source.backup -ne $false) {
    Write-Output "微信聊天记录恢复步骤："
    Write-Output "  1. 先打开微信，登录你的账号"
    Write-Output "  2. 登录后完全退出微信（托盘图标也要关掉）"
    Write-Output "  3. 告诉我'微信退出了'，我来复制数据"
    # 人类确认后：
    $wechatTarget = "$env:USERPROFILE\Documents\WeChat Files"
    robocopy "$backupPath" "$wechatTarget" /E /MT:8 /R:1 /W:1
}
```

### 浏览器书签恢复

```powershell
if ($source.type -eq "browser") {
    Write-Output "浏览器书签恢复："
    Write-Output "  最简单的方法：登录 Google/微软账号自动同步"
    Write-Output "  如果没有账号同步，告诉我，我帮你手动复制书签文件"
}
```

### 浏览器密码提醒

```powershell
# 检查 discovered_sources 中 browser 类型的 note
$browserSources = $profile.discovered_sources | Where-Object { $_.type -eq "browser" }
foreach ($bs in $browserSources) {
    if ($bs.note -match "密码未导出" -or -not $bs.passwords_exported) {
        Write-Output "提醒：浏览器保存的密码在备份时没有导出。"
        Write-Output "  重新登录各个网站时选择'记住密码'，或从旧电脑导出。"
        break
    }
}
```

---

## 第二层：开发环境恢复（只有开发者模式）

只在 `user_type = "developer"` 时执行。

### 路径替换

```powershell
function Repair-ConfigPaths {
    param([string]$Content, [string]$OldUser, [string]$NewUser)
    if ($OldUser -and $NewUser -and $OldUser -ne $NewUser) {
        $escaped = [regex]::Escape($OldUser)
        $Content = $Content -replace "(?<=[\\/])$escaped(?=[\\/`"']|\s|$)", $NewUser
    }
    return $Content
}
```

### 各开发配置恢复

按 discovered_sources 中 type="dev_config" 的数据源恢复：

```powershell
foreach ($source in ($profile.discovered_sources | Where-Object { $_.type -eq "dev_config" })) {
    switch ($source.id) {
        "gitconfig" {
            $gitconfig = Get-Content "$backupRoot\dev\gitconfig\.gitconfig" -Raw -Encoding UTF8
            $gitconfig = Repair-ConfigPaths $gitconfig $profile.system.username $env:USERNAME
            Set-Content -Path "$env:USERPROFILE\.gitconfig" -Value $gitconfig -Encoding UTF8
        }
        "ssh" {
            Copy-Item "$backupRoot\dev\ssh\*" "$env:USERPROFILE\.ssh\" -Recurse -Force
        }
        "vscode_settings" {
            $vscodeTarget = "$env:APPDATA\Code\User"
            if (-not (Test-Path $vscodeTarget)) { New-Item -ItemType Directory -Path $vscodeTarget -Force | Out-Null }
            robocopy "$backupRoot\dev\vscode" "$vscodeTarget" /E /MT:8 /R:1 /W:1
        }
        "terminal" {
            $termTarget = "$env:LOCALAPPDATA\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState"
            if (Test-Path $termTarget) {
                robocopy "$backupRoot\dev\terminal" "$termTarget" /E /MT:8 /R:1 /W:1
            }
        }
        "claude" {
            robocopy "$backupRoot\dev\claude" "$env:USERPROFILE\.claude" /E /MT:8 /R:1 /W:1
        }
    }
}
```

### 开发工具安装

需要人类配合的安装步骤：

```powershell
# Node.js（需要人类先装 NVM）
Write-Output "Node.js 安装步骤："
Write-Output "  1. 访问 https://github.com/yuruotong1/nvm-windows/releases"
Write-Output "  2. 下载安装 nvm-setup.exe"
Write-Output "  3. 装好后告诉我'NVM装好了'"
# 人类确认后：
foreach ($v in $profile.node.versions) {
    nvm install $v
    if ($LASTEXITCODE -ne 0) { Write-Output "Node $v 安装失败，跳过" }
}
nvm use $profile.node.active_version

# pnpm
npm install -g pnpm

# 全局包
foreach ($pkg in $profile.npm_global_packages) {
    npm install -g $pkg 2>$null
}

# VS Code 扩展
foreach ($ext in $profile.vscode.extensions) {
    code --install-extension $ext 2>$null
}

# Claude Code
npm install -g @anthropic-ai/claude-code
```

---

## 第三层：项目恢复（只有开发者模式）

```powershell
foreach ($repo in $profile.git_repos) {
    $workspace = $profile.system.workspace_path
    if (-not (Test-Path $workspace)) {
        $workspace = "$env:USERPROFILE\projects"
        New-Item -ItemType Directory -Path $workspace -Force | Out-Null
    }
    git clone $repo.remote "$workspace\$($repo.name)"
    if ($LASTEXITCODE -ne 0) { Write-Output "$($repo.name) clone 失败" }
}
```

---

## 恢复后审查

```powershell
Write-Output "=== 数据恢复检查 ==="

foreach ($source in $profile.discovered_sources) {
    if ($source.backup -eq $false) {
        Write-Output "⏭ $($source.name) — $($source.note)"
        continue
    }

    $target = $source.path
    $driveLetter = $target.Substring(0,1)

    if (-not (Test-Path "${driveLetter}:")) {
        $target = Get-DefaultPath -SourceId $source.id
    }

    if (Test-Path $target) {
        $count = (Get-ChildItem $target -File -ErrorAction SilentlyContinue).Count
        Write-Output "✅ $($source.name): $count 个文件 ($target)"
    } else {
        Write-Output "⚠ $($source.name): 未恢复 ($target)"
    }
}

# 开发环境检查（开发者模式）
if ($profile.user_type -eq "developer") {
    Write-Output ""
    Write-Output "=== 开发环境检查 ==="
    git --version; node --version; pnpm --version

    $gitconfig = Get-Content "$env:USERPROFILE\.gitconfig" -Raw -Encoding UTF8
    $oldUser = $profile.system.username
    $escaped = [regex]::Escape($oldUser)
    if ($gitconfig -match "(?<=[\\/])$escaped(?=[\\/`"']|\s)") {
        Write-Output "警告: .gitconfig 中仍有旧用户名 $oldUser"
    }
}
```

---

## 降级模式（没有 profile 或 profile 不含 discovered_sources）

兼容旧版备份（v1 格式，没有 discovered_sources 字段）：

```powershell
if (-not $profile.discovered_sources) {
    Write-Output "这是旧版备份格式，按旧版方式恢复。"
    Write-Output ""
    $backupRoot = "<人类指定的备份路径>"
    Write-Output "备份目录结构："
    Get-ChildItem $backupRoot -Directory | ForEach-Object {
        $count = (Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue).Count
        Write-Output "  $($_.Name): $count 个文件"
    }
    # 按 v1 逻辑恢复：personal/ → C盘, extra/ → 其他盘, dotfiles/ → 开发配置
    # ...
}
```

---

## 注意事项

- 按 discovered_sources 逐个恢复，不是按目录结构猜
- 每个数据源独立判断：原始盘在不在、有没有被格式化
- 被合并的数据源（backup: false）跳过，不重复恢复
- 路径替换用正则边界，不用简单字符串替换
- 微信恢复前必须先登录一次再退出
- 浏览器密码需要手动导出
- 未安装的工具（null/空数组）自动跳过
- 兼容旧版 v1 备份格式
