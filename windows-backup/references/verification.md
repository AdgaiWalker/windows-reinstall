# 验证脚本

第五步验证使用的 PowerShell 脚本。

## 5.2 日志分析

```powershell
$failedFiles = @()
$totalCopied = 0
$totalFailed = 0
foreach ($logFile in Get-ChildItem "$backupRoot\logs\*.log") {
    $content = Get-Content $logFile.FullName -ErrorAction SilentlyContinue
    $copied = ($content | Select-String "New File" | Measure-Object).Count
    $failed = ($content | Select-String "FAILED" | Measure-Object).Count
    $totalCopied += $copied
    $totalFailed += $failed
    if ($failed -gt 0) {
        $failedFiles += $content | Select-String "FAILED" | ForEach-Object { $_.Line.Trim() }
    }
}
Write-Output "已复制: $totalCopied 文件"
Write-Output "失败: $totalFailed 文件"
if ($failedFiles.Count -gt 0) {
    Write-Output "`n⚠️ 失败文件列表:"
    $failedFiles | ForEach-Object { Write-Output "  $_" }
}
```

FAILED > 0 时：
- 含 `Bookmarks` / `Login Data` → 浏览器被锁，建议关闭后重跑
- 含 `.lock` / `.db` → 微信/QQ 被锁
- 其他 → 列出文件名让用户判断是否重要

## 5.3 目录完整性

```powershell
foreach ($src in $backupSources) {
    $target = "$backupRoot\$($src.backup_target)"
    if (-not (Test-Path $target)) {
        Write-Output "❌ 缺失: $target"
        continue
    }
    $sourceCount = (Get-ChildItem -LiteralPath $src.Path -Recurse -File -ErrorAction SilentlyContinue | Measure-Object).Count
    $targetCount = (Get-ChildItem -LiteralPath $target -Recurse -File -ErrorAction SilentlyContinue | Measure-Object).Count
    $ratio = if ($sourceCount -gt 0) { [math]::Round($targetCount / $sourceCount * 100, 1) } else { 0 }
    if ($ratio -lt 95) {
        Write-Output "⚠️ 不完整: $($src.name) — 源 $sourceCount 文件, 备份 $targetCount 文件 ($ratio%)"
    } else {
        Write-Output "✅ $($src.name) — $targetCount 文件"
    }
}
```

文件数差异 > 5% → 列出具体差异让用户判断是否可接受。
