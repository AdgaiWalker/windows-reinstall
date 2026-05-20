# 防休眠脚本

第三步 3.1 使用。用后台进程保持系统唤醒，不修改系统电源设置。

## 为什么不改电源设置

如果修改系统电源设置（`powercfg /change standby-timeout-ac 0`），备份中途进程崩溃或被中断后，恢复代码不会执行，系统就永远不休眠了。用后台进程的方式，进程结束自动恢复。

## 启动防休眠

```powershell
# SetThreadExecutionState 每 30 秒刷新一次
$keepAwakeJob = Start-Job -ScriptBlock {
    Add-Type -TypeDefinition @'
    using System; using System.Runtime.InteropServices;
    public class Sleep { [DllImport("kernel32")] public static extern uint SetThreadExecutionState(uint es); }
'@ -PassThru | Select-Object -ExpandProperty Assembly
    while ($true) {
        [Sleep]::SetThreadExecutionState(0x80000000 -bor 0x00000002 -bor 0x00000001)  # ES_CONTINUOUS | ES_SYSTEM_REQUIRED | ES_DISPLAY_REQUIRED
        Start-Sleep -Seconds 30
    }
}
```

## 停止防休眠（备份完成后）

```powershell
Stop-Job $keepAwakeJob -ErrorAction SilentlyContinue
Remove-Job $keepAwakeJob -ErrorAction SilentlyContinue
```

进程结束后，`ES_CONTINUOUS` 标志自动失效，系统恢复原有电源设置。
