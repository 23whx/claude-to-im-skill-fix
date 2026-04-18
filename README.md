# claude-to-im-skill-fix

这个 skill 用于修复 Claude-to-IM-skill 在 Windows 上的已知安装问题。

## 背景

Claude-to-IM-skill 是一个将 Claude Code 桥接到飞书、Telegram、Discord 等 IM 平台的工具。
它的原始代码主要在 macOS/Linux 上开发和测试，在 Windows 上存在若干兼容性问题，
需要手动修补才能正常运行。

本 skill 会自动诊断并修复这些问题。

---

## 使用方法

在 Claude Code 中说：

```
/claude-to-im-skill-fix
```

或者用自然语言，例如：

- "帮我修复飞书桥接"
- "Claude-to-IM 跑不起来"
- "fix claude to im"

skill 会先诊断你的环境，列出发现的问题，经你确认后再执行修复。

---

## 已知问题清单（Windows 专项）

### 1. 核心库 `Claude-to-IM` 目录缺失

**现象**：启动时报 `Could not resolve "claude-to-im/src/lib/bridge/..."` 构建错误。

**原因**：skill 通过 `npx skills add` 安装时，只下载了 `Claude-to-IM-skill-main`（技能包装层），
但它依赖的核心库 `Claude-to-IM`（`file:../Claude-to-IM`）是一个独立仓库，
需要单独克隆到同级目录。`npm install` 会创建一个指向该目录的 junction，
但如果目录不存在，junction 就是悬空的。

**修复**：通过 GitHub API 逐文件下载核心库源码，然后 `npm install && npm run build`。
（直接 `git clone` 在某些网络环境下会被重置，GitHub API 更稳定。）

---

### 2. PowerShell 5.1 不支持 `Join-Path` 三参数

**现象**：运行 `supervisor-windows.ps1` 时报
`找不到接受实际参数"bridge.log"的位置形式参数`。

**原因**：`Join-Path path1 path2 path3` 的三参数语法是 PowerShell 6+ 才支持的特性。
Windows 内置的 PowerShell 版本是 5.1，不支持这种写法。

**修复**：将所有三参数 `Join-Path` 改为嵌套调用：
```powershell
# 修复前（PS6+）
$LogFile = Join-Path $CtiHome 'logs' 'bridge.log'

# 修复后（PS5.1 兼容）
$LogFile = Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'
```

---

### 3. `Start-Process` 不能将 stdout 和 stderr 重定向到同一文件

**现象**：启动时报 `无法将 RedirectStandardOutput 和 RedirectStandardError 同时指定为同一文件`。

**原因**：PowerShell 5.1 的 `Start-Process` 不允许 `-RedirectStandardOutput` 和
`-RedirectStandardError` 指向同一个文件路径。

**修复**：将 stderr 重定向到单独的文件 `bridge-stderr.log`：
```powershell
$StderrFile = Join-Path (Join-Path $CtiHome 'logs') 'bridge-stderr.log'
$proc = Start-Process ... -RedirectStandardOutput $LogFile -RedirectStandardError $StderrFile
```

---

### 4. `$pid` / `$Pid` 是 PowerShell 内置只读变量

**现象**：启动时报 `无法覆盖变量 PID，因为它是只读变量`。

**原因**：PowerShell 中 `$PID` 是自动变量（当前进程 ID），大小写不敏感，
因此 `$pid`、`$Pid`、`$PID` 都指向同一个只读变量，无法赋值。

**修复**：将脚本中所有用作局部变量的 `$pid` / `$Pid` 重命名为 `$bridgePid` / `$ProcessId`。

---

### 5. PowerShell supervisor 不加载 `config.env` 环境变量

**现象**：`config.env` 里设置了 `CLAUDE_CODE_GIT_BASH_PATH`，但 Claude CLI 仍然报
`Claude Code on Windows requires git-bash`。

**原因**：bash 版的 `daemon.sh` 在启动前会执行 `source config.env`，
把所有变量导出到 shell 环境，子进程自然继承。
但 PowerShell 版的 `supervisor-windows.ps1` 没有对应的步骤，
`Start-Process` 启动的 node 进程只继承当前 PowerShell 会话的环境变量，
`config.env` 里的内容根本没有进入进程环境。

**修复**：在 `start` 命令里，启动 node 进程之前，先读取 `config.env` 并注入到当前会话：
```powershell
Get-Content $ConfigFile | ForEach-Object {
    # 解析 KEY=VALUE，跳过注释和空行
    $line = $_.Trim()
    if ($line -and -not $line.StartsWith('#')) {
        $eqIdx = $line.IndexOf('=')
        if ($eqIdx -gt 0) {
            $key = $line.Substring(0, $eqIdx).Trim()
            $val = $line.Substring($eqIdx + 1).Trim().Trim('"').Trim("'")
            [System.Environment]::SetEnvironmentVariable($key, $val, 'Process')
        }
    }
}
```

---

## 开机自启动（任务计划程序）

桥接 daemon 默认是手动启动的，重启电脑后需要重新运行 `start` 命令。
本 skill 提供了一个辅助脚本，通过 Windows 任务计划程序实现登录时自动启动，无需管理员权限，无需安装额外工具。

### 安装自启动

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\skills\Claude-to-IM-skill-main\scripts\register-autostart.ps1" install
```

安装后，每次登录 Windows 时桥接 daemon 会在后台自动启动。

### 查看状态

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\skills\Claude-to-IM-skill-main\scripts\register-autostart.ps1" status
```

### 移除自启动

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\skills\Claude-to-IM-skill-main\scripts\register-autostart.ps1" uninstall
```

### 说明

- 触发条件：当前用户登录时（`AtLogOn`）
- 运行身份：当前用户（无需管理员权限）
- 启动方式：后台静默，不弹出窗口
- 如果 daemon 已在运行，`supervisor-windows.ps1 start` 会检测到并跳过，不会重复启动

---

## 环境要求

- Windows 10/11，PowerShell 5.1（系统内置版本即可）
- Node.js >= 20
- Git for Windows（提供 Git Bash，路径通常是 `C:\Program Files\Git\bin\bash.exe`
  或自定义安装路径）
- Claude Code CLI 已安装并完成认证（`claude auth login`）
- 已安装 Claude-to-IM-skill（`npx skills add op7418/Claude-to-IM-skill`）

---

## 手动修复参考

如果 skill 无法运行，也可以参考上面的问题清单手动修复。
所有修改都集中在一个文件：

```
~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1
```

核心库需要克隆到：

```
~/.claude/skills/Claude-to-IM/
```

---

## 作者 & 来源

本 skill 基于在 Windows 11 + PowerShell 5.1 环境下实际调试 Claude-to-IM-skill 的经验整理。
原始 skill 仓库：https://github.com/op7418/Claude-to-IM-skill
核心库仓库：https://github.com/op7418/Claude-to-IM