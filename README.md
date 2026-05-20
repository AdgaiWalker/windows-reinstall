# Windows Reinstall Toolkit

Windows 重装系统的备份 + 还原工具集。配合 [Claude Code](https://claude.ai/code) 使用。

## 包含两个 Skill

### windows-backup — 重装前备份向导

自动发现所有磁盘和用户数据，智能去重过滤，用 robocopy 备份到安全位置。

**核心功能：**
- 全盘扫描用户数据（桌面、文档、图片、微信、浏览器等）
- Git 仓库自动推送到远程，省去文件备份（重装后 clone 恢复）
- 开发者环境检测（SSH 密钥、.npmrc、VS Code 扩展、WSL、hosts）
- 物理磁盘感知的并行 robocopy
- 防休眠后台进程（崩溃安全）
- 备份后验证：日志 FAILED 分析 + 源/目标文件数对比

**触发方式：** 告诉 Claude "我要重装系统"、"帮我备份文件"、"C 盘满了要重装"

### windows-restore — 重装后还原向导

读取备份时生成的 `machine-profile.json`，逐个恢复数据源到原位。

**核心功能：**
- 自动定位备份盘（盘符可能变了）
- 按数据源逐个恢复，自动判断原始盘是否还在
- 开发环境恢复（Node.js、pnpm、VS Code 扩展、Git 配置）
- Git 仓库批量 clone 恢复
- 兼容旧版 v1 备份格式

**触发方式：** 告诉 Claude "重装完了帮我恢复"、"按照备份恢复"

## 快速开始

### 备份（重装前）

1. 准备一个外置硬盘或 U 盘
2. 在 Claude Code 中输入 `/windows-backup`
3. 按提示操作，等待扫描和备份完成

### 还原（重装后）

1. 装完新系统，插上备份盘
2. 在 Claude Code 中输入 `/windows-restore`
3. 按提示恢复数据和环境

## 安装

将 `windows-backup/` 和 `windows-restore/` 复制到 `~/.claude/skills/` 目录下即可。

```powershell
git clone https://github.com/AdgaiWalker/windows-reinstall.git
cp -r windows-reinstall/windows-backup ~/.claude/skills/
cp -r windows-reinstall/windows-restore ~/.claude/skills/
```

## 目录结构

```
windows-backup/
├── SKILL.md                      主流程文档
├── machine-profile.json          环境快照样例
└── references/
    ├── pitfalls.md               22 条常见陷阱
    ├── verification.md           验证脚本
    ├── disk-grouping.md          物理磁盘分组脚本
    └── keep-awake.md             防休眠脚本

windows-restore/
└── SKILL.md                      还原流程文档
```

## License

MIT
