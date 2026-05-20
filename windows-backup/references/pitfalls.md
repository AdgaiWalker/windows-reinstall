# 常见陷阱完整列表

执行 `/windows-backup` 时逐条检查。

---

## 必须遵守（违反会导致数据丢失）

1. **robocopy 必须用 PowerShell 执行** — Git Bash 会把 Windows 路径搞坏
2. **备份目标必须在受影响范围外** — 用户说全盘格式化时尤其要验证
3. **不要 git add -A** — 只提交已 track 的文件，未 track 的新文件列出来让用户决定，避免把 .env / 密钥推到远程
4. **中文路径导致 Git 仓库扫描遗漏** — `Get-ChildItem -Path -Recurse -Filter` 遇到中文路径段（如 `E:\桌面`）会静默跳过整个子树。必须用 `-LiteralPath` + 先扫已知路径（桌面、文档、用户目录），再做全盘补充
5. **Git 扫描是硬性门控** — clone-manifest.md 没生成之前不许开始备份。扫描失败必须重试，不能跳过

## 容易出错

6. **浏览器缓存占大头** — Chrome User Data 能到几十 GB，排除缓存后通常不到 2 GB
7. **浏览器密码不能直接复制文件** — 必须手动导出
8. **微信数据可能在任何盘** — 不能只搜 C 盘，搜索深度要 5 层
9. **微信/QQ 运行时文件被锁** — 提醒用户先关闭
10. **FAT32 单文件 4GB 限制** — 建议格式化为 exFAT
11. **robocopy 退出码** — 0-7 正常，8+ 才报错（1 = 有文件被复制 = 成功）
12. **重装范围不等于 C 盘** — 必须问用户哪些盘受影响
13. **不要在扫描阶段花太多时间** — 列目录名就够了，大小让 robocopy 去算
14. **备份期间防止休眠** — 长时间复制可能触发 Windows 休眠

## 容易忘

15. **SSH 密钥丢了就没了** — 重新生成密钥后要重新配所有服务器的 authorized_keys，必须备份 .ssh/
16. **.npmrc 里的私有 registry token** — 配了内网 registry 的，重装后没这个文件就连不上
17. **hosts 文件的自定义映射** — 内网域名映射丢了就忘了曾经配过什么
18. **VS Code 扩展不用备份文件** — 只要扩展列表，重装后 `code --install-extension` 批量装回来
19. **WSL 发行版可能很大** — export 前先问用户，几十 GB 的发行版可以选择不导出，重装后从 Microsoft Store 重装
20. **Steam 游戏可跳过但存档要保留** — SteamLibrary\userdata 里是存档
21. **Git 远程仓库不需要备份** — 有远程且已推送的，clone 即可恢复
22. **不要硬编码 C:\Users\...\Desktop** — 桌面可能重定向到其他盘，用 `[Environment]::GetFolderPath()`
