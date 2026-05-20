# Windows Reinstall Toolkit

Windows 重装系统的备份 + 还原工具集。配合 [Claude Code](https://claude.ai/code) 使用。

## 设计理念

**大多数数据要么能重新下载，要么备份了也恢复不了。真正不可替代的可能只有几个 GB。**

v3 不再试图备份所有东西，而是帮你快速搞清楚哪些不用备份，用最短时间保护不可替代的数据。

## 两个 Skill

### [windows-backup](windows-backup/SKILL.md) — 重装前备份

- **快速赢家**（5-10 分钟）：git push 所有仓库、导出浏览器书签、拷贝 SSH 密钥
- **磁盘类型自动检测**：SSD → 4 路并行压缩，HDD → 串行，USB 闪存盘 → 先压到本地再拷贝
- **动态路径**：桌面可能在 E 盘，用 Windows API 检测实际位置，不硬编码
- **Git 自动同步**：检测到 git 就自动扫描所有仓库，推送到远程后排除备份（重装后 clone 恢复）
- **路径去重**：桌面和下载指向同一目录时只备份一次
- **云上传压缩**：可选，90 万文件压成几个分卷，传网盘快 10-100 倍

**触发：** 告诉 Claude "我要重装系统"、"帮我备份文件"、"C 盘满了要重装"

### [windows-restore](windows-restore/SKILL.md) — 重装后还原

- 读取 `machine-profile.json`，按数据源逐个恢复
- 自动判断原始盘是否还在、是否被格式化
- Git 仓库按 action 区分：clone 恢复 vs 文件复制 vs 跳过
- 压缩包自动发现 + 分卷解压（同样检测磁盘类型选并行/串行）
- 兼容旧版 v1 备份格式

**触发：** 告诉 Claude "重装完了帮我恢复"、"按照备份恢复"

## 快速开始

### 备份（重装前）

1. 准备一个外置硬盘或 U 盘
2. 在 Claude Code 中输入 `/windows-backup`
3. 先跑快速赢家（5 分钟），再按需全量备份

### 还原（重装后）

1. 装完新系统，插上备份盘
2. 在 Claude Code 中输入 `/windows-restore`
3. 按提示恢复数据和环境

## 安装

将 `windows-backup/` 和 `windows-restore/` 复制到 `~/.claude/skills/` 目录下。

```powershell
git clone https://github.com/AdgaiWalker/windows-reinstall.git
Copy-Item -Recurse windows-reinstall\windows-backup "$env:USERPROFILE\.claude\skills\"
Copy-Item -Recurse windows-reinstall\windows-restore "$env:USERPROFILE\.claude\skills\"
```

## 目录结构

```
windows-reinstall/
├── README.md
├── LICENSE
├── windows-backup/
│   └── SKILL.md
└── windows-restore/
    └── SKILL.md
```

## 版本历史

| 版本 | 变化 |
|------|------|
| v3 | 快速通道重写、磁盘类型检测、并行压缩策略、路径去重 |
| v2.1 | PowerShell 全重写、动态路径、git 自动同步 |
| v1 | 初始版本（Git Bash + 硬编码路径） |

## License

MIT
