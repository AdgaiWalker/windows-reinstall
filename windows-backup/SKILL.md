---
name: windows-backup
description: >
  Windows 重装前的备份向导。先搞清楚哪些不用备份，再快速处理真正不可替代的数据。
  当用户提到"重装系统"、"备份文件"、"重装前备份"、"C盘满了要重装"、
  "帮我找出需要备份的文件"、"换电脑"、"迁移环境"时触发。
  任何人都能用，不需要懂技术。
---

# Windows 重装前备份向导

帮人在重装系统前快速搞清楚：哪些不用备份，哪些必须备份。目标是用最短时间保护不可替代的数据。

## 核心原则

**大多数数据要么能重新下载，要么备份了也恢复不了。真正不可替代的可能只有几个 GB。**

不要试图备份所有东西。按优先级处理：

1. **秒级操作** — git push、导出浏览器书签、拷贝 SSH 密钥
2. **分钟级操作** — 个人文档、照片拷到 U 盘
3. **可选操作** — 微信迁移、全量备份（用户明确要求才做）

**所有命令必须用 PowerShell 工具执行。** robocopy 在 Git Bash 下路径解析失败。

---

## 第零步：环境评估

### 0.0 管理员权限检查

```powershell
$isAdmin = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
if (-not $isAdmin) { Write-Output "WARNING: 当前不是管理员权限，部分文件可能无法访问。" }
```

### 0.1 列出所有磁盘

```powershell
Get-CimInstance Win32_LogicalDisk | ForEach-Object {
    $used = [math]::Round(($_.Size - $_.FreeSpace) / 1GB, 1)
    $free = [math]::Round($_.FreeSpace / 1GB, 1)
    $total = [math]::Round($_.Size / 1GB, 1)
    "$($_.DeviceID) | $($_.VolumeName) | Total: ${total} GB | Used: ${used} GB | Free: ${free} GB | $($_.FileSystem)"
}
```

### 0.2 问用户：重装会影响哪些盘？

用 AskUserQuestion 展示磁盘列表。选项：只格 C 盘 / 部分盘 / 全盘格式化 / 自己指定。

从回答得到 `$affectedDrives`，例如 `@("C:\", "D:\")`。

### 0.3 确认备份目标

列出不在受影响范围内的盘作为候选。**强制校验**：备份盘不能在受影响盘列表里。FAT32 → 警告单文件 4GB 限制。

---

## 第一步：快速赢家（5-10 分钟，价值最高）

**先做这些，再做任何扫描。** 这些操作加起来不到 10 分钟，但保护的数据比后面几小时的 robocopy 更有价值。每项都是"先发现，再执行"，不跳步。

### 1.1 Git 仓库扫描与同步（有 git 就自动执行）

**发现：** 不问用户是不是开发者。直接检测：

```powershell
$hasGit = $null -ne (Get-Command git -ErrorAction SilentlyContinue)
```

没有 git → 跳过。有 git → 扫描受影响盘上的所有仓库：

```powershell
$repos = @()
$excludeDirs = @("node_modules", ".cache", ".pnpm-store", "__pycache__", ".next", "vendor")
foreach ($drive in $affectedDrives) {
    Get-ChildItem -Path $drive -Directory -Recurse -Depth 4 -Filter ".git" -ErrorAction SilentlyContinue |
        Where-Object { $excludeDirs -notcontains $_.Parent.Name } |
        ForEach-Object {
            $repoPath = $_.Parent.FullName
            $remote = git -C $repoPath remote get-url origin 2>$null
            $branch = git -C $repoPath rev-parse --abbrev-ref HEAD 2>$null
            $uncommitted = (git -C $repoPath status --porcelain 2>$null | Measure-Object).Count
            $unpushed = 0
            if ($remote) { $unpushed = (git -C $repoPath log "origin/$branch..HEAD" --oneline 2>$null | Measure-Object).Count }
            $repos += [PSCustomObject]@{
                Path = $repoPath; Remote = $remote; Branch = $branch
                Uncommitted = $uncommitted; Unpushed = $unpushed
            }
        }
}
# git 命令可能返回非零退出码，不影响结果，重置退出码
exit 0
```

分类展示并处理：

**执行：** 对"需要推送"的仓库，逐个问用户是否推送。用户确认后：

```
Git 仓库扫描结果：

✅ 已同步（不需要备份，重装后 git clone 恢复）：
  blog — 25 MB → github.com/xxx/blog

⚠️ 需要推送（本地有远程没有的代码）：
  desk — 12 GB, 11 个未推送提交 + 5 个未提交文件

❗ 无远程（纯本地仓库，必须文件备份）：
  my-tool — 200 MB
```

对"需要推送"的仓库，逐个问用户是否推送。确认后执行：

```powershell
git -C $repoPath add -A
if ((git -C $repoPath status --porcelain | Measure-Object).Count -gt 0) {
    git -C $repoPath commit -m "pre-reinstall backup"
}
git -C $repoPath push origin $branch
```

推送成功 → `action: clone`（备份时跳过，重装后 clone 恢复）。失败或无远程 → `action: backup`。

**关键：构建 robocopy 排除列表。** `action: clone` 的仓库，整个目录都不需要 robocopy 备份（不只是 `.git`）。按备份源分组，收集每个源路径下需要排除的子目录名：

```powershell
# $repos 是 1.1 扫描的结果，每个有 Path, Remote, Action
# 按备份源（如 E:\桌面）分组，收集需要排除的子目录
$repoExcludeBySource = @{}
foreach ($repo in $repos | Where-Object { $_.Action -eq 'clone' }) {
    # 找到这个仓库属于哪个备份源
    foreach ($source in $backupSources) {
        if ($repo.Path -like "$($source.Path)\*" -or $repo.Path -eq $source.Path) {
            if (-not $repoExcludeBySource[$source.Path]) {
                $repoExcludeBySource[$source.Path] = @()
            }
            # 如果仓库路径就是源路径本身（如桌面根本身是 git 仓库），跳过——备份源本身就是仓库时不能排除自己
            if ($repo.Path -ne $source.Path) {
                $relativeDir = $repo.Path.Replace("$($source.Path)\", "")
                $repoExcludeBySource[$source.Path] += $relativeDir
            }
        }
    }
}
# 示例结果：$repoExcludeBySource["E:\桌面"] = @("blog", "set-UX", "skills", "转化记录", "AdgaiWalker")
```

将此列表传给第三步 robocopy 的 `/XD` 参数，避免重复备份已有远程的仓库。

### 1.2 浏览器书签导出（30 秒）

检测已安装的浏览器，提醒用户手动导出书签和密码：
```powershell
$userProfile = $env:USERPROFILE
$browsers = @()
if (Test-Path "$userProfile\AppData\Local\Google\Chrome\User Data") { $browsers += "Chrome" }
if (Test-Path "$userProfile\AppData\Local\Microsoft\Edge\User Data") { $browsers += "Edge" }
if (Test-Path "$userProfile\AppData\Roaming\Mozilla\Firefox\Profiles") { $browsers += "Firefox" }
```

**告诉用户：** 浏览器密码不能直接复制文件恢复，必须手动导出（设置 → 密码 → 导出）。书签同理。这步 30 秒，但如果不做就丢了。

### 1.3 SSH 密钥 + 开发配置（1 分钟）

```powershell
$sshDir = "$env:USERPROFILE\.ssh"
$gitconfig = "$env:USERPROFILE\.gitconfig"
$npmrc = "$env:USERPROFILE\.npmrc"
$claudeDir = "$env:USERPROFILE\.claude"

foreach ($item in @(@("SSH密钥", $sshDir), @("Git配置", $gitconfig), @("npm配置", $npmrc), @("Claude配置", $claudeDir))) {
    if (Test-Path $item[1]) { Write-Output "$($item[0]) = $($item[1])" }
}
```

SSH 密钥丢了 = 服务器登不上。这是优先级最高的文件。

---

## 第二步：发现不可替代的数据

### 2.1 Windows 特殊文件夹（动态检测，不硬编码）

```powershell
$folders = [ordered]@{
    'Desktop'   = [Environment]::GetFolderPath('Desktop')
    'Documents' = [Environment]::GetFolderPath('MyDocuments')
    'Pictures'  = [Environment]::GetFolderPath('MyPictures')
    'Music'     = [Environment]::GetFolderPath('MyMusic')
    'Videos'    = [Environment]::GetFolderPath('MyVideos')
    'Downloads' = $null
}
$dl = (Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders" -Name "{374DE290-123F-4565-9164-39C4925E467B}" -ErrorAction SilentlyContinue).'{374DE290-123F-4565-9164-39C4925E467B}'
if ($dl) { $folders['Downloads'] = [Environment]::ExpandEnvironmentVariables($dl) }

foreach ($name in $folders.Keys) {
    $path = $folders[$name]
    if ($path -and (Test-Path $path)) { Write-Output "$name = $path" }
}
```

### 2.2 应用数据（微信、QQ、钉钉）

**只搜受影响盘，不搜备份盘。** 搜索深度 5 层。

```powershell
$patterns = @("WeChat Files", "xwechat_files", "Tencent Files", "DingTalk", "wxid_*")
foreach ($drive in $affectedDrives) {
    foreach ($pattern in $patterns) {
        Get-ChildItem -Path $drive -Directory -Recurse -Depth 5 -Filter $pattern -ErrorAction SilentlyContinue |
            ForEach-Object { Write-Output $_.FullName }
    }
}
```

**微信处理建议：** 推荐用户使用微信自带的"迁移到另一台设备"功能（WiFi 直传），比拷文件恢复更可靠。如果用户坚持文件备份，正常 robocopy。

### 2.3 列出受影响盘目录名

```powershell
foreach ($drive in $affectedDrives) {
    if (Test-Path $drive) {
        Write-Output "=== $drive ==="
        Get-ChildItem -Path $drive -Directory -Name
    }
}
```

### 2.4 自动分级 + 用户确认

| 级别 | 匹配规则 |
|------|---------|
| 🔴 必备 | Desktop, Documents, WeChat Files, xwechat_files, Tencent Files, .ssh, .claude, .gitconfig, 项目目录, 自媒体 |
| 🟡 建议 | 软件包, 安装包, obs录屏, 录屏, 工具, workspace, 备份 |
| ⚪ 可跳 | SteamLibrary, steam, .pnpm-store, node_modules, .cache, game, 游戏, QQMusicCache |

展示分级结果，用 AskUserQuestion 让用户确认或调整。

**向用户展示预估：**
```
数据分级结果：

🔴 不可替代（丢了真没了）：~5 GB
  个人文档、照片、SSH 密钥

🟡 建议备份（重装后可能有用）：~15 GB
  安装包、录屏、工具

⚪ 可跳过（能重新下载或备份也恢复不了）：~300 GB
  AI 模型 → 可从 CivitAI/HuggingFace 重新下载
  Steam 游戏 → 可重新下载
  浏览器缓存 → 重装后自动重建
```

---

## 第三步：去重 + 执行备份

### 3.0 路径去重（备份前必须做）

收集第二步所有数据源，路径归一化（去末尾斜杠、统一大小写）：

- **完全重复** → 合并，子项标记 `backup: false`
- **子目录关系** → 父目录覆盖，子目录标记 `backup: false`

例如：桌面路径 `E:\桌面` 和下载路径也指向 `E:\桌面` → 只备份一次。

### 3.1 备份前准备

### 3.1 备份前准备

```powershell
$backupRoot = "$backupTarget\Windows-Backup-$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path "$backupRoot\personal", "$backupRoot\dev", "$backupRoot\extra", "$backupRoot\logs" -Force

# 防止休眠（用注册表 GUID 读取，不依赖系统语言）
$activeScheme = powercfg /getactivescheme | ForEach-Object { if ($_ -match '([a-fA-F0-9-]{36})') { $Matches[1] } }
$standbyValue = (Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Power\User\PowerSchemes\$activeScheme\Sub_sleep\STANDBYIDLE" -Name "ACSettingIndex" -ErrorAction SilentlyContinue).ACSettingIndex
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
```

**提醒用户关闭微信/QQ/浏览器（文件可能被锁）。**

### 3.2 执行 robocopy

<<<<<<< HEAD
对每个 `backup: true` 的数据源并行执行：

```powershell
robocopy.exe "<源路径>" "$backupRoot\<backup_target>" /E /MT:8 /R:2 /W:1 /XD node_modules .next .cache .pnpm-store dist build __pycache__ <.git（仅 action=clone 的仓库）> <类型专属排除> /XF Thumbs.db desktop.ini .DS_Store *.tmp *.temp ~$* *.bak *.dmp *.mdmp *.crash /NP /TEE /LOG:"$backupRoot\logs\<id>.log"
=======
对每个 `backup: true` 的数据源并行执行。**必须排除 `action: clone` 的仓库整个目录**（不只是 `.git`）：

```powershell
# 通用排除
$commonXD = @("node_modules", ".next", ".cache", ".pnpm-store", "dist", "build", "__pycache__")

# 对每个备份源，合并通用排除 + 该源下的 clone 仓库目录
foreach ($source in $backupSources | Where-Object { $_.backup }) {
    $xdArgs = $commonXD.Clone()
    if ($repoExcludeBySource[$source.Path]) {
        $xdArgs += $repoExcludeBySource[$source.Path]
    }
    # 如果备份源本身不是 git 仓库，也排除 .git（避免子目录的 .git 被复制）
    # 如果备份源本身就是 action=clone 的仓库，排除 .git 但保留工作文件
    $xdArgs += ".git"

    robocopy.exe $source.Path "$backupRoot\$($source.backup_target)" /E /MT:8 /R:2 /W:1 /XD $xdArgs /XF Thumbs.db desktop.ini .DS_Store *.tmp *.temp ~$* *.bak *.dmp *.mdmp *.crash /NP /TEE /LOG:"$backupRoot\logs\$($source.id).log"
}
>>>>>>> cd05ce9 (feat: 排除有远程仓库的目录，避免重复备份)
```

退出码 0-7 正常，≥8 报错。浏览器数据源额外排除 `Cache`, `Code Cache`, `GPUCache`, `Service Worker`, `blob_storage`, `IndexedDB`。

### 3.3 锁定文件检查

robocopy 完成后检查日志：FAILED 文件含 `Bookmarks` 或 `Login Data` → 警告用户关闭浏览器重跑。其他被锁文件 → 列出但允许继续。

### 3.4 恢复休眠

```powershell
if ($standbyValue) {
    powercfg /setacvalueindex SCHEME_CURRENT SUB_SLEEP STANDBYIDLE $standbyValue
} else {
    powercfg /change standby-timeout-ac 15
}
```

---

## 第四步：云上传压缩（可选）

如果用户要传网盘，先把零碎文件打包再传，90 万个文件变成几个大文件，上传速度快 10-100 倍。

### 4.1 自动检测磁盘类型，选择并行策略

```powershell
# 检测源盘和目标盘是 SSD 还是 HDD
function Get-DiskType {
    param([string]$DriveLetter)
    $letter = $DriveLetter.Substring(0,1)
    $disk = Get-Partition -DriveLetter $letter -ErrorAction SilentlyContinue |
        Get-Disk -ErrorAction SilentlyContinue
    if ($disk) { return $disk.ProvisioningType }  # "Fixed" = 通常SSD
    # 备用方案：通过物理磁盘检测
    $physicalDisk = Get-PhysicalDisk | Where-Object {
        $_.DeviceId -in (Get-Partition -DriveLetter $letter -ErrorAction SilentlyContinue | Get-Disk -ErrorAction SilentlyContinue).Number
    }
    return if ($physicalDisk) { $physicalDisk.MediaType } else { "Unknown" }
}

# 判断策略：源盘或目标盘任一是 HDD → 串行；都是 SSD → 并行
$sourceType = Get-DiskType $sourceDrive  # 备份数据所在盘
$targetType = Get-DiskType $targetDrive  # 压缩包输出盘
$useParallel = ($sourceType -eq "SSD" -and $targetType -eq "SSD")
$maxWorkers = if ($useParallel) { 4 } else { 1 }
Write-Output "磁盘类型: 源=$sourceType, 目标=$targetType → $($maxWorkers)路$($useParallel ? '并行' : '串行')压缩"
```

**原理：** HDD 机械硬盘磁头只能在一个位置，4 路同时读写 = 磁头疯狂跳来跳去，比串行更慢。SSD 没有机械部件，并行读写效率高。

**U 盘/外接盘特殊处理：** `MediaType = Unspecified` + `BusType = USB` 通常是闪存盘。随机 IO 弱，4 路并行写反而慢。最优策略：**先压到本地 SSD（并行），压完再串行拷贝到 U 盘**。压缩阶段吃 SSD 性能，拷贝阶段只走一次 USB 带宽。

```powershell
# 检测 USB 闪存盘
$isUSB = (Get-PhysicalDisk | Where-Object { $_.BusType -eq "USB" }).Count -gt 0
if ($isUSB -and $useParallel) {
    Write-Output "检测到 USB 外接盘：先压到本地 SSD（并行），再拷贝到外接盘（串行）"
    $compressTarget = $localSSDDrive  # 压缩输出到本地 SSD
} else {
    $compressTarget = $targetDrive    # 直接压到目标盘
}
```

### 4.2 执行压缩

按分类分别压缩，每组独立输出：

```powershell
# 分组定义
$groups = @(
    @{ Name = "wechat";       Source = "$backupRoot\personal\wechat";       Archive = "01-wechat" }
    @{ Name = "desktop-docs"; Source = "$backupRoot\personal\desktop","$backupRoot\personal\documents"; Archive = "02-desktop-docs" }
    @{ Name = "browser-dev";  Source = "$backupRoot\personal\browser-bookmarks","$backupRoot\dev"; Archive = "03-browser-dev" }
    @{ Name = "zimeiti";      Source = "$backupRoot\personal\自媒体","$backupRoot\personal\card"; Archive = "04-zimeiti" }
)
```

**WinRAR（推荐，大多数 Windows 用户已安装）：**
```powershell
foreach ($group in $groups) {
    & "C:\Program Files\WinRAR\Rar.exe" a -s -m3 -v4000m -r "$outputDir\$($group.Archive).rar" $group.Source
    # 并行模式下用 Start-Job 或后台 PowerShell 同时跑多个组
}
```

**7-Zip（压缩率更高）：**
```powershell
foreach ($group in $groups) {
    & "C:\Program Files\7-Zip\7z.exe" a -mx=5 -v4000m "$outputDir\$($group.Archive).7z" $group.Source
}
```

**并行执行（仅 SSD）：** 多个组用 `Start-Job` 或 `run_in_background` 同时跑。串行模式（HDD）逐个跑，避免磁头争抢。

| 格式 | 压缩率 | 速度 | 固实压缩 | 分卷 | 推荐 |
|------|--------|------|---------|------|------|
| 7z | 最好 | 最慢 | 有 | 支持 | 小文件多时优先选 |
| RAR | 中等 | 中等 | 有 | 支持 | 已装 WinRAR 时选 |
| ZIP | 最差 | 最快 | 无 | 不支持 | 不推荐 |

### 4.3 解压策略（重装后恢复）

**同样自动检测磁盘类型。** 压缩包各自解压到独立目录（wechat → personal\wechat, desktop → personal\desktop），目录不重叠，并行解压安全。

```powershell
# 检测解压目标盘类型
$targetType = Get-DiskType $restoreDrive
$maxWorkers = if ($targetType -eq "SSD") { 4 } else { 1 }
# 并行解压每个分卷，互不干扰
```

---

## 第五步：生成 machine-profile.json + 文档

### machine-profile.json 核心字段

```json
{
  "generated_at": "ISO 8601",
  "system": { "username": "", "os": "", "special_folders": {} },
  "disks": [{ "drive_letter": "", "total_gb": 0, "used_gb": 0, "affected": true }],
  "discovered_sources": [{
    "id": "", "type": "", "name": "", "path": "", "size_gb": 0,
    "backup": true, "backup_target": "", "note": ""
  }],
  "git_repos": [{
    "path": "", "remote_url": "", "branch": "", "action": "clone|backup|skip"
  }],
  "backup_meta": { "backup_root_path": "", "estimated_total_gb": 0 },
  "browser": { "profiles": [] },
  "ssh": { "has_keys": false },
  "vscode": { "extensions": [] }
}
```

原子写入：先写 `.tmp`，验证 JSON 格式正确再 rename。

### 备份清单.md + 还原指南.md

还原指南按 git action 区分：`clone` 的仓库直接 `git clone -b <branch> <url> <path>`，`backup` 的从备份目录复制。

---

## 第六步：验证

1. 解析 `machine-profile.json` 确认格式正确
2. 每个 `backup: true` 的数据源：备份目录存在、文件数 > 0
3. 检查备份盘总使用量

---

## 常见陷阱

1. **robocopy 必须用 PowerShell 执行** — Git Bash 会把 Windows 路径搞坏
2. **不要硬编码 C:\Users\...\Desktop** — 桌面可能重定向到其他盘
3. **备份目标必须在受影响范围外** — 全盘格式化时尤其要验证
4. **浏览器缓存占大头** — Chrome User Data 能到几十 GB，排除缓存后不到 2 GB
5. **微信数据可能在任何盘** — 搜索深度要 5 层
6. **微信/QQ 运行时文件被锁** — 提醒用户先关闭
7. **浏览器密码不能直接复制文件** — 必须手动导出
8. **FAT32 单文件 4GB 限制** — 建议格式化为 exFAT
9. **robocopy 退出码** — 0-7 正常，8+ 才报错（1 = 有文件被复制 = 成功）
10. **重装范围不等于 C 盘** — 必须问用户哪些盘受影响
<<<<<<< HEAD
11. **Git 远程仓库不需要备份** — 有远程且已推送的，clone 即可恢复
=======
11. **Git 远程仓库不需要备份** — 有远程且已推送的，整个目录都不用 robocopy（不只是 .git），重装后 `git clone` 恢复。第一步扫描后按备份源分组收集排除列表，第三步 robocopy 通过 `/XD` 跳过
>>>>>>> cd05ce9 (feat: 排除有远程仓库的目录，避免重复备份)
12. **不要在扫描阶段花太多时间** — 列目录名就够了，大小让 robocopy 去算
13. **备份期间防止休眠** — 长时间复制可能触发 Windows 休眠
14. **Steam 游戏可跳过但存档要保留** — SteamLibrary\userdata 里是存档
15. **大多数数据不需要备份** — AI 模型可重下、浏览器缓存可重建、VS Code 配置有同步功能。先做 git push 和导出书签，这俩 5 分钟搞定，价值比几小时的 robocopy 大
