# 物理磁盘分组脚本

第三步 3.2 使用。按物理盘分组数据源，同盘串行，跨盘并行。

```powershell
# 建立物理磁盘映射（逻辑盘 → 物理盘编号）
$diskMap = @{}
Get-CimInstance Win32_LogicalDiskToPartition | ForEach-Object {
    $drive = $_.Antecedent.DeviceID  # 如 "C:"
    $disk = $_.Dependent.DiskIndex   # 如 0
    if ($drive) { $diskMap[$drive] = $disk }
}

# 按物理盘分组数据源
$groups = @{}
foreach ($src in $backupSources) {
    $driveLetter = $src.Path.Substring(0,2)
    $diskId = if ($diskMap.ContainsKey($driveLetter)) { $diskMap[$driveLetter] } else { 99 }
    if (-not $groups.ContainsKey($diskId)) { $groups[$diskId] = @() }
    $groups[$diskId] += $src
}

# 每个物理盘组内串行，组间并行
# 对 $groups.Keys 中的每个 $diskId，组内 foreach 串行执行 robocopy
# 不同 $diskId 的组可以并行
```

## 为什么要分组

两块物理盘的场景：
- Disk 0 (WD SN740): C:, D:, E:
- Disk 1 (SIX SSD): F:, G:, H:

如果在 Disk 0 上并行跑两个 robocopy（备份 C:\Documents 和 D:\软件），两路 IO 争抢同一块盘的带宽，速度反而比串行慢。不同物理盘上的源（如 E:\ 和 F:\）可以安全并行。
