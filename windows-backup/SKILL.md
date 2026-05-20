---
name: windows-backup
description: >
  Windows 重装前的备份扫描向导。自动发现所有磁盘，检测用户类型（普通人/开发者），
  全盘发现用户数据源并智能去重过滤，生成 machine-profile.json 环境镜像。
  当用户提到"要重装系统"、"系统备份"、"备份配置"、"重装前备份"、"C盘满了要重装"、
  "帮我找出需要备份的文件"、"换电脑"、"迁移环境"时触发。
  支持任何人使用：不需要懂技术，能看懂中文就行。
---

# Windows 重装前备份向导

帮人类在重装系统前找出所有重要文件。你不知道自己的数据散落在哪 — 这就是你要用这个 skill 的原因。

## 核心设计

整个备份流程分三个阶段，严格分离：

1. **发现** — 无脑全扫，所有盘上能找到的用户数据都找出来
2. **处理** — 去重 + 过滤垃圾 + 计算真实大小
3. **备份** — 用户确认后的干净数据，直接复制

不要把发现和备份混在一起。

---

## 第零步：全盘发现

**这是最重要的一步。** 不要假设数据只在 C 盘。

### 0.1 列出所有磁盘

```powershell
Get-Volume | Where-Object { $_.DriveLetter } | ForEach-Object {
    $usedGB = [math]::Round(($_.Size - $_.SizeRemaining) / 1GB, 1)
    $totalGB = [math]::Round($_.Size / 1GB, 1)
    Write-Output "$($_.DriveLetter): | $($_.FileSystemLabel) | $usedGB GB / $totalGB GB | $($_.FileSystemType)"
}
```

### 0.2 问用户：重装会影响哪些盘？

用 AskUserQuestion 问：
```
你的电脑有以下磁盘：
  C: Win11Pro — 233 GB / 250 GB (NTFS) ← 系统盘
  D: 软件 — 149 GB / 227 GB (NTFS)
  E: 办公学习 — 63 GB / 300 GB (NTFS)
  ...

重装系统会影响哪些盘？（只格C盘？还是全盘格式化？）
```

选项示例：
- 只格 C 盘（系统盘）
- C 盘 + D 盘
- 全盘格式化
- 我自己指定

### 0.3 扫描受影响盘的目录结构

对用户选定的每个盘，扫描 top-level 目录并计算大小：

```powershell
foreach ($drive in $affectedDrives) {
    Write-Output "`n=== ${drive}: ==="
    Get-ChildItem "${drive}\" -Directory -ErrorAction SilentlyContinue | ForEach-Object {
        $size = [math]::Round(((Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
        if ($size -gt 0.05) { Write-Output "  $($_.Name): $size GB" }
    }
}
```

**注意：** 这步可能很慢（大目录），用 `run_in_background` 并行扫描多个盘。

### 0.4 自动分级 + 用户确认

根据目录名和内容自动分级：

| 级别 | 标记 | 包含内容 | 匹配规则 |
|------|------|---------|---------|
| 必备 | `🔴` | 个人文档、聊天记录、密钥、项目代码、配置文件 | Desktop, Documents, WeChat Files, Tencent Files, .ssh, .claude, .gitconfig, 项目目录 |
| 建议 | `🟡` | 软件安装包、录屏、开发工具、重要下载 | 软件包, obs录屏, 录屏, 安装包, 工具, workspace, 备份 |
| 可跳 | `⚪` | Steam游戏、缓存、node_modules、可重新下载的 | SteamLibrary, steam, .pnpm-store, node_modules, .cache, game, 游戏 |

展示给用户：
```
受影响盘的目录（自动分级，你可以调整）：

🔴 必备（约 XX GB）：
  E:\桌面: 12 GB — 你的桌面文件
  E:\新建文件夹: 41 GB — 看起来是重要项目

🟡 建议备份（约 XX GB）：
  D:\Windows安装包: 6 GB — 系统安装包，重装后可能有用

⚪ 可跳过（约 XX GB）：
  D:\SteamLibrary: 34 GB — 游戏可重新下载

总计：必备 XX GB + 建议 XX GB = XX GB
```

用 AskUserQuestion 让用户确认或调整分级。如果总大小超过备份盘空间，提示按优先级砍。

---

## 第一步：发现数据源（全盘扫描，不管哪个盘受影响）

这一步只负责"找到"，不做任何过滤和判断。

### 1.1 Windows 特殊文件夹

用 Windows API 获取实际路径，支持文件夹重定向：

```powershell
$sources = @()

# 桌面
$desktop = [Environment]::GetFolderPath("Desktop")
if (Test-Path $desktop) {
    $size = [math]::Round(((Get-ChildItem $desktop -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="desktop"; type="special_folder"; name="桌面"; path=$desktop; size_gb=$size; backup_target="personal\desktop" }
}

# 文档
$documents = [Environment]::GetFolderPath("MyDocuments")
if (Test-Path $documents) {
    $size = [math]::Round(((Get-ChildItem $documents -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="documents"; type="special_folder"; name="文档"; path=$documents; size_gb=$size; backup_target="personal\documents" }
}

# 下载（用 Shell API，支持重定向）
$shell = New-Object -ComObject Shell.Application
$downloads = $shell.Namespace('shell:Downloads').Self.Path
if (Test-Path $downloads) {
    $size = [math]::Round(((Get-ChildItem $downloads -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="downloads"; type="special_folder"; name="下载"; path=$downloads; size_gb=$size; backup_target="personal\downloads" }
}

# 图片
$pictures = [Environment]::GetFolderPath("MyPictures")
if (Test-Path $pictures) {
    $size = [math]::Round(((Get-ChildItem $pictures -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="pictures"; type="special_folder"; name="图片"; path=$pictures; size_gb=$size; backup_target="personal\pictures" }
}
```

### 1.2 应用数据（微信、QQ 等）

全盘搜索，不限受影响的盘。数据可能在任何盘上：

```powershell
# 所有本地盘（排除 USB 外接盘）
$usbDriveLetters = Get-Disk | Where-Object { $_.BusType -eq 'USB' } |
    Get-Partition | Where-Object { $_.DriveLetter } |
    ForEach-Object { $_.DriveLetter }
$allDrives = Get-Volume | Where-Object { $_.DriveLetter -and ($usbDriveLetters -notcontains $_.DriveLetter) } |
    ForEach-Object { "$($_.DriveLetter):" }

# 微信数据
$wechatPath = $null
foreach ($drive in $allDrives) {
    $found = Get-ChildItem "${drive}\" -Directory -Recurse -Depth 3 -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -match "^(WeChat Files|xwechat_files|wxid_)" -and $_.Name -notmatch "^\." }
    if ($found) { $wechatPath = $found[0].FullName; break }
}
if ($wechatPath -and (Test-Path $wechatPath)) {
    $size = [math]::Round(((Get-ChildItem $wechatPath -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="wechat"; type="app_data"; name="微信聊天记录"; path=$wechatPath; size_gb=$size; backup_target="personal\wechat" }
}

# QQ 数据
$qqPath = $null
foreach ($drive in $allDrives) {
    $found = Get-ChildItem "${drive}\" -Directory -Recurse -Depth 3 -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -match "^Tencent Files" }
    if ($found) { $qqPath = $found[0].FullName; break }
}
if ($qqPath -and (Test-Path $qqPath)) {
    $size = [math]::Round(((Get-ChildItem $qqPath -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
    $sources += @{ id="qq"; type="app_data"; name="QQ聊天记录"; path=$qqPath; size_gb=$size; backup_target="personal\qq" }
}
```

### 1.3 浏览器数据

```powershell
# Chrome
$chromeRoot = "$env:LOCALAPPDATA\Google\Chrome\User Data"
if (Test-Path $chromeRoot) {
    $profiles = Get-ChildItem $chromeRoot -Directory -ErrorAction SilentlyContinue |
        Where-Object { Test-Path "$($_.FullName)\Bookmarks" }
    if ($profiles) {
        $size = [math]::Round(((Get-ChildItem $chromeRoot -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
        $sources += @{ id="chrome"; type="browser"; name="Chrome 数据"; path=$chromeRoot; size_gb=$size; backup_target="personal\browser-bookmarks\chrome"; profiles=($profiles | ForEach-Object { $_.Name }) }
    }
}

# Edge
$edgeRoot = "$env:LOCALAPPDATA\Microsoft\Edge\User Data"
if (Test-Path $edgeRoot) {
    $profiles = Get-ChildItem $edgeRoot -Directory -ErrorAction SilentlyContinue |
        Where-Object { Test-Path "$($_.FullName)\Bookmarks" }
    if ($profiles) {
        $size = [math]::Round(((Get-ChildItem $edgeRoot -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
        $sources += @{ id="edge"; type="browser"; name="Edge 数据"; path=$edgeRoot; size_gb=$size; backup_target="personal\browser-bookmarks\edge"; profiles=($profiles | ForEach-Object { $_.Name }) }
    }
}
```

### 1.4 开发者数据（检测到开发工具才扫描）

先检测用户类型：

```powershell
$devTools = @(
    @{ Name = "Git"; Test = "git --version" },
    @{ Name = "Node.js"; Test = "node --version" },
    @{ Name = "Python"; Test = "python --version" },
    @{ Name = "VS Code"; Test = "code --version" },
    @{ Name = "Docker"; Test = "docker --version" }
)
$devCount = 0
foreach ($tool in $devTools) {
    try { Invoke-Expression $tool.Test 2>$null | Out-Null; $devCount++ } catch {}
}
$isDeveloper = $devCount -ge 2
```

如果是开发者，额外扫描：

```powershell
if ($isDeveloper) {
    # Git 配置
    $gitconfig = "$env:USERPROFILE\.gitconfig"
    if (Test-Path $gitconfig) {
        $sources += @{ id="gitconfig"; type="dev_config"; name="Git 配置"; path=$gitconfig; size_gb=0; backup_target="dev\gitconfig" }
    }

    # SSH
    $sshDir = "$env:USERPROFILE\.ssh"
    if (Test-Path $sshDir) {
        $hasKeys = Get-ChildItem $sshDir -File -ErrorAction SilentlyContinue | Where-Object { $_.Name -match "id_" }
        $sources += @{ id="ssh"; type="dev_config"; name="SSH 配置"; path=$sshDir; size_gb=0; backup_target="dev\ssh"; has_keys=($hasKeys.Count -gt 0) }
    }

    # VS Code 设置
    $vscodeDir = "$env:APPDATA\Code\User"
    if (Test-Path $vscodeDir) {
        $sources += @{ id="vscode_settings"; type="dev_config"; name="VS Code 设置"; path=$vscodeDir; size_gb=0; backup_target="dev\vscode" }
    }

    # Terminal 设置
    $terminalDir = "$env:LOCALAPPDATA\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState"
    if (Test-Path $terminalDir) {
        $sources += @{ id="terminal"; type="dev_config"; name="Terminal 设置"; path=$terminalDir; size_gb=0; backup_target="dev\terminal" }
    }

    # Claude Code
    $claudeDir = "$env:USERPROFILE\.claude"
    if (Test-Path $claudeDir) {
        $size = [math]::Round(((Get-ChildItem $claudeDir -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB), 1)
        $sources += @{ id="claude"; type="dev_config"; name="Claude Code 配置"; path=$claudeDir; size_gb=$size; backup_target="dev\claude" }
    }

    # npm 配置
    $npmrc = "$env:USERPROFILE\.npmrc"
    if (Test-Path $npmrc) {
        $sources += @{ id="npmrc"; type="dev_config"; name="npm 配置"; path=$npmrc; size_gb=0; backup_target="dev\npmrc" }
    }
}
```

### 1.5 Git 远程仓库检测（开发者模式）

**只有检测到 Git 才执行这一步。** 很多开发者的项目目录里有大量 git 仓库，如果远程仓库已经有完整代码，重装后 `git clone` 就能恢复，不需要浪费备份空间。

**核心规则：有远程仓库 + 代码已推送 = 不需要备份。**

扫描所有数据源路径下的 git 仓库：

```powershell
if ($isDeveloper) {
    $gitRepos = @()
    $scanPaths = $sources | Where-Object { $_.type -in @("special_folder", "custom") } | ForEach-Object { $_.path }

    foreach ($scanPath in $scanPaths) {
        if (-not (Test-Path $scanPath)) { continue }

        $gitDirs = Get-ChildItem $scanPath -Directory -Recurse -Depth 3 -ErrorAction SilentlyContinue |
            Where-Object { Test-Path "$($_.FullName)\.git" -PathType Container }

        foreach ($dir in $gitDirs) {
            $remoteUrl = $null
            try { $remoteUrl = git -C $dir.FullName remote get-url origin 2>$null } catch {}

            if (-not $remoteUrl) { continue }  # 无远程仓库，正常备份

            $branch = git -C $dir.FullName branch --show-current 2>$null
            $status = git -C $dir.FullName status --porcelain 2>$null
            $hasUncommitted = ($status -and $status.Count -gt 0)

            $ahead = @()
            if ($branch) {
                try { $ahead = git -C $dir.FullName log --oneline "origin/$branch..HEAD" 2>$null } catch {}
            }
            $hasUnpushed = ($ahead -and $ahead.Count -gt 0)

            $size = [math]::Round(((Get-ChildItem $dir.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1MB), 0)

            $gitRepos += @{
                path = $dir.FullName
                remote_url = $remoteUrl
                branch = $branch
                has_uncommitted = $hasUncommitted
                has_unpushed = $hasUnpushed
                size_mb = $size
                status = if ($hasUncommitted -or $hasUnpushed) { "dirty" } else { "clean" }
            }
        }
    }

    # 按大小排序，大的优先展示
    $gitRepos = $gitRepos | Sort-Object { $_.size_mb } -Descending
}
```

向用户展示结果，分三类：

```
Git 仓库扫描结果：

✅ 可跳过（远程已有，重装后 clone 即可）：
  E:\桌面\blog — 25 MB → github.com/AdgaiWalker/AdgaiWalker.git
  E:\桌面\set-UX — 3 MB → github.com/AdgaiWalker/set-app.git
  E:\桌面\skills — 10 MB → github.com/MiniMax-AI/skills.git (第三方)
  ...

⚠️ 需要先 push（有未提交/未推送的更改）：
  E:\桌面 (desk.git) — 12434 MB → github.com/AdgaiWalker/desk.git
    11 个未推送提交 + 337 个未提交文件
  E:\桌面\项目\NorthStar — 411 MB → github.com/AdgaiWalker/NorthStar.git
    3 个未提交文件
  ...

✅ 已是干净状态，直接跳过（同上）
```

用 AskUserQuestion 让用户选择处理方式：

```
选项：
  先 push 所有 dirty 仓库，然后全部跳过备份（推荐）
  dirty 仓库直接备份目录（不 push）
  我自己选哪些 push，哪些备份
```

**用户确认后，记录到 `$gitRepos` 变量中**，后续步骤使用：
- `status = "clean"` 的仓库 → 从备份数据源中排除
- `status = "dirty"` 但用户已 push → 更新为 `"clean"`，然后排除
- 用户选择直接备份的 dirty 仓库 → 保留在数据源中，但标记 `git_repo = true`

**减去的备份数量要展示给用户：**
```
Git 仓库优化：跳过 XX 个仓库，节省约 XX GB
  github.com/AdgaiWalker/desk.git — 12.4 GB
  github.com/MiniMax-AI/skills.git × 3 — 47 MB
  ...

仍需备份的 Git 仓库：XX 个，约 XX GB
```

### 1.6 自定义目录（来自第零步用户确认的目录）

第零步中用户确认的 🔴必备 和 🟡建议 目录，如果在第一步没有被其他数据源覆盖，也要加入数据源列表：

```powershell
# 收集第一步已覆盖的路径
$coveredPaths = $sources | ForEach-Object { $_.path.TrimEnd('\') }

# 把第零步确认的目录加入数据源
$customId = 0
foreach ($dir in $confirmedDirs) {
    # 跳过 ⚪可跳 的
    if ($dir.priority -eq "skip") { continue }

    $normalizedDir = $dir.path.TrimEnd('\')

    # 检查是否已被第一步的数据源覆盖（完全匹配或父路径匹配）
    $isCovered = $false
    foreach ($covered in $coveredPaths) {
        if ($normalizedDir -eq $covered -or $normalizedDir.StartsWith("$covered\")) {
            $isCovered = $true
            break
        }
    }
    if ($isCovered) { continue }

    $dirName = Split-Path $dir.path -Leaf
    $sources += @{
        id = "custom_$customId"
        type = "custom"
        name = $dirName
        path = $dir.path
        size_gb = $dir.size_gb
        backup_target = "extra\$($dir.path.Substring(0,1))_$dirName"
        priority = $dir.priority
    }
    $customId++
}
```

---

## 第二步：智能处理（去重 + 过滤 + 计算真实大小）

### 2.1 路径去重

检查数据源之间是否有路径重叠：

```powershell
# 去重逻辑
$processedSources = @()
$seenPaths = @{}  # path -> source id

foreach ($source in $sources) {
    $normalizedPath = $source.path.TrimEnd('\')

    if ($seenPaths.ContainsKey($normalizedPath)) {
        # 路径重复，合并
        $existingId = $seenPaths[$normalizedPath]
        # 在已处理的源上标注合并信息
        $idx = [array]::IndexOf(($processedSources | ForEach-Object { $_.id }), $existingId)
        if ($idx -ge 0) {
            $existing = $processedSources[$idx]
            if (-not $existing.merged_with) { $existing.merged_with = @() }
            $existing.merged_with += $source.id
            $existing.note = "路径与 $($source.name) 相同，已合并备份"
        }
        continue
    }

    # 检查是否是已有路径的子目录
    $isChild = $false
    foreach ($parentPath in $seenPaths.Keys) {
        if ($normalizedPath.StartsWith("$parentPath\")) {
            $isChild = $true
            $parentIdx = [array]::IndexOf(($processedSources | ForEach-Object { $_.id }), $seenPaths[$parentPath])
            if ($parentIdx -ge 0) {
                if (-not $processedSources[$parentIdx].contains) { $processedSources[$parentIdx].contains = @() }
                $processedSources[$parentIdx].contains += $source.id
            }
            break
        }
    }
    if ($isChild) {
        # 子目录不单独备份，父目录会覆盖
        $source.backup = $false
        $source.note = "包含在另一个数据源中"
    }

    $seenPaths[$normalizedPath] = $source.id
    $processedSources += $source
}

$sources = $processedSources
```

### 2.2 Git 仓库排除

根据第一步 1.5 的扫描结果，将已 push 到远程的 git 仓库从备份中排除：

```powershell
# 收集所有已确认可跳过的 git 仓库路径
$skippedGitPaths = $gitRepos | Where-Object { $_.status -eq "clean" } | ForEach-Object { $_.path.TrimEnd('\') }

# 从数据源中减去 git 仓库的大小
foreach ($source in $sources) {
    if ($source.backup -eq $false) { continue }

    $sourcePath = $source.path.TrimEnd('\')
    $skippedSize = 0

    foreach ($gitPath in $skippedGitPaths) {
        # git 仓库在此数据源内
        if ($gitPath.StartsWith("$sourcePath\")) {
            $gitSizeBytes = (Get-ChildItem $gitPath -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum
            $skippedSize += $gitSizeBytes
        }
    }

    if ($skippedSize -gt 0) {
        $skippedGB = [math]::Round($skippedSize / 1GB, 1)
        $source.filtered_size_gb = [math]::Round($source.size_gb - $skippedGB, 1)
        $source.git_skipped_gb = $skippedGB
        $source.note = if ($source.note) { "$($source.note)；排除 $skippedGB GB git 仓库（可 clone 恢复）" } else { "排除 $skippedGB GB git 仓库（可 clone 恢复）" }
    }
}
```

将跳过的 git 仓库列表写入 profile 的 `git_repos` 字段，恢复时直接 clone：

```json
"git_repos": [
    {
        "path": "E:\\桌面\\blog",
        "remote_url": "https://github.com/AdgaiWalker/AdgaiWalker.git",
        "branch": "main",
        "size_mb": 25,
        "status": "clean"
    }
]
```

恢复时（windows-restore skill）对这些仓库执行 `git clone -b <branch> <remote_url> <path>`。

### 2.3 垃圾过滤

每种数据源类型有对应的排除规则：

| 数据源类型 | 默认排除 | 原因 |
|-----------|---------|------|
| `browser` | Cache, Code Cache, GPUCache, Service Worker, blob_storage, IndexedDB | 可重新生成，占空间大 |
| `special_folder` | Thumbs.db, desktop.ini | 系统缓存文件 |
| `dev_config` | node_modules, .next, .cache, .pnpm-store | 开发缓存，可重新安装 |
| `app_data` | 无 | 聊天记录每个文件都有价值 |

### 2.4 计算过滤后的真实大小

```powershell
foreach ($source in $sources) {
    if ($source.backup -eq $false) { continue }

    if ($source.type -eq "browser") {
        # 浏览器：排除缓存后重新计算大小
        $allFiles = Get-ChildItem $source.path -Recurse -File -ErrorAction SilentlyContinue |
            Where-Object { $_.DirectoryName -notmatch '(\\|/)(Cache|Code Cache|GPUCache|Service Worker|blob_storage|IndexedDB)(\\|/|$)' }
        $source.filtered_size_gb = [math]::Round(($allFiles | Measure-Object -Property Length -Sum).Sum / 1GB, 1)
        $source.cache_removed_gb = [math]::Round($source.size_gb - $source.filtered_size_gb, 1)
        $source.note = "已排除 $($source.cache_removed_gb) GB 缓存"
    } else {
        $source.filtered_size_gb = $source.size_gb
    }
}
```

### 2.5 垃圾扫描（发现可能不需要的文件）

任何人的电脑用久了都会积累垃圾。这一步帮用户找出并排除它们，让备份干净，恢复时也不会把垃圾带回来。

**A. 自动排除（robocopy 直接跳过）**

这些是任何电脑上都有的垃圾，不需要问用户：

| 模式 | 谁都会有 |
|------|---------|
| `Thumbs.db`、`desktop.ini`、`.DS_Store` | 系统自动生成的缩略图和配置 |
| `*.tmp`、`*.temp`、`~$*`、`*.bak` | 软件产生的临时文件 |
| `*.dmp`、`*.mdmp`、`*.crash` | 程序崩溃产生的转储文件 |
| `*.log`（超过 30 天的） | 软件运行日志 |

在第五步 robocopy 时，所有数据源统一排除这些，不按类型区分。

**B. 标记提醒（扫出来让用户决定）**

判断标准只有一个：**大 + 旧 = 可能不需要**。不依赖文件类型或技术知识。

```powershell
$junkCandidates = @()
$sixMonthsAgo = (Get-Date).AddMonths(-6)
$threeMonthsAgo = (Get-Date).AddMonths(-3)

foreach ($source in $sources) {
    if ($source.backup -eq $false) { continue }

    $files = Get-ChildItem $source.path -Recurse -File -ErrorAction SilentlyContinue

    foreach ($file in $files) {
        $sizeMB = [math]::Round($file.Length / 1MB, 0)
        $age = ((Get-Date) - $file.LastWriteTime).Days

        # 规则 1：超过 100MB 且 6 个月没动过
        if ($file.Length -gt 100MB -and $file.LastWriteTime -lt $sixMonthsAgo) {
            $junkCandidates += @{
                path = $file.FullName
                size_mb = $sizeMB
                age_days = $age
                reason = "$sizeMB MB，$([math]::Round($age/30)) 个月没动过"
                source_name = $source.name
            }
        }

        # 规则 2：下载文件夹里超过 3 个月没动过的
        if ($source.id -eq "downloads" -and $file.LastWriteTime -lt $threeMonthsAgo -and $file.Length -gt 10MB) {
            $junkCandidates += @{
                path = $file.FullName
                size_mb = $sizeMB
                age_days = $age
                reason = "下载文件，$sizeMB MB，$([math]::Round($age/30)) 个月前下载的"
                source_name = $source.name
            }
        }
    }

    # 空目录
    $emptyDirs = Get-ChildItem $source.path -Recurse -Directory -ErrorAction SilentlyContinue |
        Where-Object { (Get-ChildItem $_.FullName -File -Recurse -ErrorAction SilentlyContinue).Count -eq 0 }
    if ($emptyDirs.Count -gt 5) {
        $junkCandidates += @{
            path = "($($emptyDirs.Count) 个空文件夹)"
            size_mb = 0
            age_days = 0
            reason = "$($emptyDirs.Count) 个空文件夹，可能是残留"
            source_name = $source.name
        }
    }
}

# 按大小排序，只展示 top 20
$junkCandidates = $junkCandidates | Sort-Object { [int]($_.size_mb) } -Descending | Select-Object -First 20
$junkTotalGB = [math]::Round(($junkCandidates | Measure-Object { $_.size_mb } -Sum).Sum / 1024, 1)
```

向用户展示（用日常语言，不用术语）：
```
扫描发现这些文件你可能不需要了：

⚠ 大文件，很久没动过（共 $junkTotalGB GB）：
  E:\桌面\old_backup_2024.zip — 2.1 GB，8 个月没动过
  E:\桌面\录屏_2024-03.mp4 — 1.2 GB，1 年没动过
  C:\...\Downloads\setup_v3.exe — 890 MB，6 个月前下载的
  ...（共 XX 个）

要不要排除这些？（排除 = 不备份，重装后也不会恢复）

选项：
  全部排除（推荐）
  我自己选
  全部保留
```

用户确认后，把排除的文件记到 profile 里，恢复时按这个列表跳过。

**C. 数据分布报告（让用户看到空间花在哪了）**

用所有人都能看懂的分类：

```powershell
foreach ($source in $sources) {
    if ($source.backup -eq $false) { continue }

    $files = Get-ChildItem $source.path -Recurse -File -ErrorAction SilentlyContinue

    $typeGroups = $files | Group-Object {
        $ext = $_.Extension.ToLower()
        switch -Regex ($ext) {
            '^\.(zip|rar|7z|tar|gz)$' { "压缩包" }
            '^\.(mp4|avi|mkv|mov|wmv|flv)$' { "视频" }
            '^\.(jpg|jpeg|png|gif|bmp|svg|webp|heic)$' { "图片" }
            '^\.(exe|msi|iso|dmg|app)$' { "安装包" }
            '^\.(pdf|doc|docx|xls|xlsx|ppt|pptx|txt|md)$' { "文档" }
            '^\.(mp3|wav|flac|aac|ogg|wma)$' { "音乐" }
            '^\.(db|sqlite|sqlite3)$' { "数据库" }
            default { if ($ext) { $ext } else { "其他" } }
        }
    } | Sort-Object { ($_.Group | Measure-Object -Property Length -Sum).Sum } -Descending | Select-Object -First 6

    $source.type_distribution = $typeGroups | ForEach-Object {
        $size = [math]::Round(($_.Group | Measure-Object -Property Length -Sum).Sum / 1GB, 1)
        @{ type=$_.Name; count=$_.Count; size_gb=$size }
    }
}
```

展示给用户：
```
你的数据长什么样：

桌面 (E:\桌面) — 11.4 GB：
  视频    4.5 GB (12 个)
  压缩包  3.2 GB (28 个)
  图片    2.1 GB (156 个)
  文档    1.0 GB (89 个)
  其他    0.6 GB

文档 (C:\...\Documents) — 3.7 GB：
  文档    1.8 GB (234 个)
  图片    1.2 GB (89 个)
  其他    0.7 GB
```

### 2.6 展示处理结果

向用户展示去重和过滤后的结果：
```
数据源发现结果：

✅ 桌面 (E:\桌面) — 11.4 GB
✅ 文档 (C:\...\Documents) — 3.7 GB
✅ 微信聊天记录 (F:\xwechat_files) — 14 GB
🔗 下载 → 与桌面路径相同，已合并
✅ Chrome 数据 — 4.2 GB（已排除 14.3 GB 缓存）
✅ QQ聊天记录 — 1.7 GB

过滤后总计：XX GB（原始 XX GB，去重和过滤省了 XX GB）
```

---

## 第三步：生成 machine-profile.json

```json
{
  "generated_at": "<ISO 8601>",
  "generator": "windows-backup skill v2",
  "user_type": "<developer|general>",
  "system": {
    "username": "<当前用户名>",
    "os": "<系统版本>",
    "workspace_path": "<工作区路径>"
  },
  "disks": [
    {
      "drive_letter": "<盘符>",
      "label": "<卷标>",
      "filesystem": "<文件系统>",
      "total_gb": 0,
      "used_gb": 0,
      "affected_by_reinstall": true
    }
  ],
  "affected_drives": ["<受影响的盘符列表>"],
  "discovered_sources": [
    {
      "id": "<唯一标识>",
      "type": "<special_folder|app_data|browser|dev_config|custom>",
      "name": "<中文名称>",
      "path": "<实际路径>",
      "backup_target": "<备份目录下的相对路径>",
      "size_gb": 0,
      "filtered_size_gb": 0,
      "backup": true,
      "merged_with": [],
      "contains": [],
      "excluded_files": [],
      "type_distribution": [{"type":"","count":0,"size_gb":0}],
      "note": ""
    }
  ],
  "browser": {
    "chrome_profiles": ["<列表>"],
    "edge_profiles": ["<列表>"]
  },
  "deep_scan": {
    "large_files": [{"path":"<路径>","size_gb":0}],
    "database_files": [{"path":"<路径>","size_mb":0}],
    "license_files": ["<列表>"],
    "has_wsl": false,
    "has_docker": false
  },
  "git": { "user_name": "", "user_email": "" },
  "ssh": { "has_keys": false, "note": "" },
  "node": { "versions": [], "active_version": "", "npm_prefix": "", "npm_cache": "" },
  "python": { "version": "" },
  "vscode": { "extensions": [], "has_snippets": false },
  "npm_global_packages": [],
  "git_repos": [
    {
      "path": "<原始本地路径>",
      "remote_url": "<远程仓库 URL>",
      "branch": "<分支名>",
      "size_mb": 0,
      "status": "<clean|dirty>",
      "restored_via_clone": true
    }
  ],
  "backup_meta": {
    "backup_root_path": "<实际路径>",
    "disk_volume_label": "<卷标>",
    "estimated_total_gb": 0,
    "source_paths": {}
  }
}
```

`discovered_sources` 是核心新字段。每个数据源独立记录，restore skill 按这个列表恢复。

`extra_directories` 字段已合并进 `discovered_sources`（非 C 盘的数据也是数据源，不需要单独处理）。

**原子写入**（防断电损坏）：
```powershell
$jsonContent | Set-Content "$backupRoot\machine-profile.json.tmp" -Encoding UTF8
try { Get-Content "$backupRoot\machine-profile.json.tmp" -Raw | ConvertFrom-Json | Out-Null } catch { Write-Error "JSON 验证失败" }
Move-Item "$backupRoot\machine-profile.json.tmp" "$backupRoot\machine-profile.json" -Force
```

---

## 第四步：确认备份目标

### 4.1 自动识别外接盘

```powershell
$usbDrives = @()
$usbDisks = Get-Disk | Where-Object { $_.BusType -eq 'USB' }
foreach ($disk in $usbDisks) {
    $partitions = $disk | Get-Partition | Where-Object { $_.DriveLetter }
    foreach ($part in $partitions) {
        $letter = $part.DriveLetter
        $vol = Get-Volume -DriveLetter $letter -ErrorAction SilentlyContinue
        $freeGB = [math]::Round($vol.SizeRemaining / 1GB, 1)
        $totalGB = [math]::Round($vol.Size / 1GB, 1)
        $usbDrives += @{
            DriveLetter = "$letter"
            Label = $vol.FileSystemLabel
            FreeGB = $freeGB
            TotalGB = $totalGB
            FileSystem = $vol.FileSystemType
        }
    }
}
```

- 找到 1 个 → 直接确认
- 找到多个 → 列出来让用户选
- 没找到 → 提示插入外接盘，或让用户手动指定盘符

### 4.2 检查文件系统

```powershell
$volume = Get-Volume -DriveLetter $driveLetter -ErrorAction SilentlyContinue
if ($volume.FileSystemType -eq "FAT32") {
    Write-Output "你的U盘是FAT32格式，单文件不能超过4GB。"
    Write-Output "建议：把U盘格式化为exFAT（右键U盘→格式化→选exFAT）"
}
```

### 4.3 空间检查

展示过滤后大小 vs 可用空间，不够时按优先级砍（先砍⚪可跳，再砍🟡建议）。

---

## 第五步：执行备份

### 备份目录结构

```
<备份目录>\
├── machine-profile.json
├── personal\              ← Windows 特殊文件夹 + 应用数据
│   ├── desktop\
│   ├── documents\
│   ├── downloads\         ← 如果和桌面路径相同，跳过
│   ├── pictures\
│   ├── wechat\
│   ├── qq\
│   └── browser-bookmarks\
│       ├── chrome\
│       └── edge\
├── dev\                   ← 开发者配置（只有开发者模式）
│   ├── gitconfig\
│   ├── ssh\
│   ├── vscode\
│   ├── terminal\
│   ├── claude\
│   └── npmrc\
├── dotfiles\              ← 其他 dotfile
├── skills\
├── npm-global-list\
└── dev-env-info\
```

### 复制前提醒

- 微信/QQ 在运行 → 提示关闭（文件被锁）
- Chrome/Edge 在运行 → 提示关闭（书签文件被锁）

### robocopy 命令（排除列表根据数据源类型动态生成）

```powershell
# 通用垃圾排除（所有数据源都适用）
$globalJunkDirs = @("node_modules", ".next", ".cache", ".pnpm-store", "dist", "build", "__pycache__")
$globalJunkFiles = @("Thumbs.db", "desktop.ini", ".DS_Store", "*.tmp", "*.temp", "~$*", "*.bak", "*.dmp", "*.mdmp", "*.crash")

# 按数据源类型追加额外排除
switch ($source.type) {
    "browser" { $extraDirs = @("Cache", "Code Cache", "GPUCache", "Service Worker", "blob_storage", "IndexedDB") }
    default   { $extraDirs = @() }
}

# Git 仓库排除：已 push 到远程的 git 仓库不需要备份
$gitExcludeDirs = @()
$skippedGitRepos = $gitRepos | Where-Object { $_.status -eq "clean" }
foreach ($repo in $skippedGitRepos) {
    # 如果 git 仓库在此数据源内，排除其目录名
    $repoDirName = Split-Path $repo.path -Leaf
    $repoParent = Split-Path $repo.path -Parent
    if ($source.path.TrimEnd('\').StartsWith($repoParent.TrimEnd('\')) -or $repoParent.TrimEnd('\').StartsWith($source.path.TrimEnd('\'))) {
        $gitExcludeDirs += $repoDirName
    }
}

$allExcludeDirs = $globalJunkDirs + $extraDirs + $gitExcludeDirs | Select-Object -Unique
$excludeArgs = @()
if ($allExcludeDirs.Count -gt 0) { $excludeArgs += ($allExcludeDirs | ForEach-Object { "/XD $_" }) }
if ($globalJunkFiles.Count -gt 0) { $excludeArgs += ($globalJunkFiles | ForEach-Object { "/XF $_" }) }
$excludeStr = $excludeArgs -join " "

robocopy "$source" "$target" /E /MT:8 /R:1 /W:1 $excludeStr
```

检查退出码：`if ($LASTEXITCODE -ge 8) { 报错 }`（0-7 都是正常的）

### 跳过被合并的数据源

如果某个数据源的 `backup` 为 `false`（被去重合并了），跳过它的 robocopy。在 profile 里标注了 `merged_with` 和 `note`，restore 时知道去哪找。

---

## 第六步：验证

```powershell
$backupRoot = "<备份目录>"

# 1. 验证 profile JSON
try { $profile = Get-Content "$backupRoot\machine-profile.json" -Raw -Encoding UTF8 | ConvertFrom-Json }
catch { Write-Error "profile 损坏: $($_.Exception.Message)" }

# 2. 根据 discovered_sources 逐个检查
foreach ($source in $profile.discovered_sources) {
    if ($source.backup -eq $false) {
        Write-Output "⏭ $($source.name) ($($source.id)) — 已合并或跳过"
        continue
    }

    # 找到对应的备份子目录
    $backupPath = Join-Path $backupRoot $source.backup_target
    if (-not (Test-Path $backupPath)) {
        Write-Output "⚠ $($source.name) — 备份目录不存在！"
        continue
    }

    $fileCount = (Get-ChildItem $backupPath -Recurse -File -ErrorAction SilentlyContinue).Count
    $backupSizeMB = [math]::Round(((Get-ChildItem $backupPath -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1MB), 0)

    if ($fileCount -eq 0) {
        Write-Output "⚠ $($source.name) — 空目录，可能遗漏"
    } else {
        Write-Output "✅ $($source.name) — $fileCount 个文件, $backupSizeMB MB"
    }
}

# 3. 总备份大小
$totalMB = [math]::Round(((Get-ChildItem $backupRoot -Recurse -File -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1MB), 0)
Write-Output "`n总备份: $totalMB MB"
```

---

## 第七步：生成文档

1. **备份清单.md** — 备份了什么，每个数据源多大，来自哪个盘，哪些被合并/过滤了
2. **还原指南.md** — 重装后怎么恢复（参照 discovered_sources 逐个恢复）

文档中要包含：
- 每个数据源的来源路径和过滤情况
- 去重说明（哪些路径被合并了）
- 垃圾过滤说明（浏览器排除了多少缓存）
- 哪些盘被格式化了，哪些没动

---

## 常见陷阱

1. **只扫 C 盘是最大的坑** — 很多用户数据在 D/E/F 盘
2. **路径去重必须做** — Windows 允许多个特殊文件夹指向同一个目录
3. **浏览器缓存占大头** — Chrome User Data 能到几十 GB，大部分是缓存
4. **微信数据可能在任何盘** — 不能只在 C 盘找
5. **微信/QQ 运行时文件被锁** — 备份前先关闭
6. **浏览器密码不能直接复制文件** — 必须手动导出
7. **FAT32 单文件 4GB 限制** — 建议格式化为 exFAT
8. **robocopy 退出码** — 0-7 正常，8+ 才报错
9. **中文用户名路径** — 所有文件操作用 UTF-8 编码
10. **重装范围不等于 C 盘** — 必须问用户哪些盘受影响
11. **数据源发现和备份要分离** — 发现时全扫，备份时按需
12. **Git 远程仓库不需要备份** — 有远程仓库且代码已推送的项目，重装后 `git clone` 即可恢复，能省几十 GB。必须检查是否有未提交/未推送的更改，先 push 再跳过
