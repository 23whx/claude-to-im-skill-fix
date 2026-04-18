---
name: claude-to-im-skill-fix
description: 修复 Claude-to-IM-skill 在 Windows 上的已知安装和兼容性问题。当用户说 /claude-to-im-skill-fix、"帮我修复飞书桥接"、"Claude-to-IM 跑不起来"、"fix claude to im" 等时触发。
---

# Claude-to-IM-skill-fix

你是 Claude-to-IM-skill 的 Windows 修复助手。

Skill 目录（SKILL_DIR）在 `~/.claude/skills/claude-to-im-skill-fix/`。
README 文件在 `SKILL_DIR/README.md`，包含所有已知问题的详细说明，供参考。

Claude-to-IM-skill 的安装目录（TARGET_DIR）通常在 `~/.claude/skills/Claude-to-IM-skill-main/`。
核心库目录（CORE_DIR）在 `~/.claude/skills/Claude-to-IM/`。

---

## 执行流程

### 第一步：诊断

运行以下检查，收集结果：

1. **检查 TARGET_DIR 是否存在**
   - 路径：`~/.claude/skills/Claude-to-IM-skill-main/`
   - 不存在则提示用户先安装：`npx skills add op7418/Claude-to-IM-skill`

2. **检查 CORE_DIR 是否存在且有效**
   - 路径：`~/.claude/skills/Claude-to-IM/`
   - 检查 `CORE_DIR/src/lib/bridge/context.ts` 是否存在
   - 检查 `CORE_DIR/dist/lib/bridge/context.js` 是否存在（已构建）

3. **检查 supervisor-windows.ps1 的 PS5.1 兼容性**
   - 文件路径：`TARGET_DIR/scripts/supervisor-windows.ps1`
   - 检查是否存在三参数 `Join-Path`（pattern：`Join-Path \S+ '[^']+' '[^']+'`）
   - 检查是否存在 `-RedirectStandardOutput $LogFile` 和 `-RedirectStandardError $LogFile` 同时出现
   - 检查是否存在 `\$pid\b` 或 `\$Pid\b` 作为局部变量（非函数参数）
   - 检查 `start` 命令块中是否有读取 `config.env` 并注入环境变量的代码

4. **检查 config.env**
   - 路径：`~/.claude-to-im/config.env`
   - 检查是否存在
   - 检查是否设置了 `CLAUDE_CODE_GIT_BASH_PATH`
   - 如果没有，检查系统上 bash.exe 的实际位置（`where bash`）

5. **检查开机自启动任务**
   - 运行 `Get-ScheduledTask -TaskName 'ClaudeToIMBridge' -ErrorAction SilentlyContinue`
   - 未找到则标记为 ⚠️（可选修复）

6. **检查 Node.js 版本**
   - 运行 `node --version`，需要 >= 20

7. **检查 Claude CLI**
   - 运行 `claude --version`，需要 >= 2.x

### 第二步：汇报诊断结果

用清晰的表格或列表展示每项检查的结果：
- ✅ 正常
- ❌ 有问题（说明具体问题）
- ⚠️ 警告（可选修复）

如果所有检查都通过，告知用户环境正常，不需要修复。

### 第三步：询问用户确认

列出需要修复的项目，使用 AskUserQuestion 询问用户是否继续执行修复。

### 第四步：执行修复

按顺序修复每个问题，每修复一项就告知用户进度。

---

## 修复方法

### 修复 A：下载并构建核心库 Claude-to-IM

当 `~/.claude/skills/Claude-to-IM/` 不存在或不完整时执行。

**步骤：**

1. 创建目录结构：
```bash
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/adapters
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/markdown
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/security
```

2. 通过 GitHub API 逐文件下载（直接 git clone 在某些网络环境会被重置，API 更稳定）：

对每个文件，调用：
```
GET https://api.github.com/repos/op7418/Claude-to-IM/contents/{path}
```
响应中的 `content` 字段是 base64 编码的文件内容，用 PowerShell 解码并写入：
```powershell
$resp = Invoke-RestMethod "https://api.github.com/repos/op7418/Claude-to-IM/contents/$path"
$bytes = [System.Convert]::FromBase64String(($resp.content -replace '\n',''))
[System.IO.File]::WriteAllBytes("$dest\$path", $bytes)
```

需要下载的文件列表：
- `package.json`
- `tsconfig.json`
- `tsconfig.build.json`
- `src/lib/bridge/types.ts`
- `src/lib/bridge/host.ts`
- `src/lib/bridge/context.ts`
- `src/lib/bridge/channel-adapter.ts`
- `src/lib/bridge/channel-router.ts`
- `src/lib/bridge/bridge-manager.ts`
- `src/lib/bridge/conversation-engine.ts`
- `src/lib/bridge/delivery-layer.ts`
- `src/lib/bridge/permission-broker.ts`
- `src/lib/bridge/adapters/index.ts`
- `src/lib/bridge/adapters/feishu-adapter.ts`
- `src/lib/bridge/adapters/telegram-adapter.ts`
- `src/lib/bridge/adapters/telegram-media.ts`
- `src/lib/bridge/adapters/telegram-utils.ts`
- `src/lib/bridge/adapters/discord-adapter.ts`
- `src/lib/bridge/adapters/qq-adapter.ts`
- `src/lib/bridge/adapters/qq-api.ts`
- `src/lib/bridge/markdown/ir.ts`
- `src/lib/bridge/markdown/render.ts`
- `src/lib/bridge/markdown/feishu.ts`
- `src/lib/bridge/markdown/telegram.ts`
- `src/lib/bridge/markdown/discord.ts`
- `src/lib/bridge/security/rate-limiter.ts`
- `src/lib/bridge/security/validators.ts`

3. 安装依赖并构建：
```bash
cd ~/.claude/skills/Claude-to-IM && npm install && npm run build
```

4. 重新安装 skill 依赖（刷新 junction）：
```bash
cd ~/.claude/skills/Claude-to-IM-skill-main && npm install
```

### 修复 B：修复 supervisor-windows.ps1 的 PS5.1 兼容性

文件路径：`~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1`

**B1 — 修复三参数 Join-Path**

将：
```powershell
$LogFile    = Join-Path $CtiHome 'logs' 'bridge.log'
$DaemonMjs  = Join-Path $SkillDir 'dist' 'daemon.mjs'
```
改为：
```powershell
$LogFile    = Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'
$DaemonMjs  = Join-Path (Join-Path $SkillDir 'dist') 'daemon.mjs'
```

**B2 — 修复 stdout/stderr 重定向到同一文件**

在路径定义区域添加 `$StderrFile`，并修改 `Start-Process` 调用：
```powershell
$StderrFile = Join-Path (Join-Path $CtiHome 'logs') 'bridge-stderr.log'
```
将 `-RedirectStandardError $LogFile` 改为 `-RedirectStandardError $StderrFile`。

**B3 — 修复 $pid 只读变量冲突**

- 将函数参数 `param([string]$Pid)` 改为 `param([string]$ProcessId)`，函数体内同步替换
- 将局部变量 `$pid = Start-Fallback` 改为 `$bridgePid = Start-Fallback`
- 将局部变量 `$pid = Read-Pid`（stop/status 命令块中）改为 `$bridgePid = Read-Pid`，后续引用同步替换

**B4 — 添加 config.env 环境变量注入**

在 `start` 命令块的 `else` 分支（fallback 启动路径）中，`Start-Fallback` 调用之前插入：

```powershell
# 添加 $ConfigFile 路径变量（在文件顶部路径定义区域）
$ConfigFile = Join-Path $CtiHome 'config.env'

# 在 start 命令的 else 分支中，Start-Fallback 之前插入：
if (Test-Path $ConfigFile) {
    Get-Content $ConfigFile | ForEach-Object {
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
}
[System.Environment]::SetEnvironmentVariable('CLAUDECODE', $null, 'Process')
```

### 修复 C：设置 CLAUDE_CODE_GIT_BASH_PATH

当 `config.env` 中没有 `CLAUDE_CODE_GIT_BASH_PATH` 时执行。

1. 用 `where bash` 找到 bash.exe 路径
2. 将以下行添加到 `~/.claude-to-im/config.env`：
```
CLAUDE_CODE_GIT_BASH_PATH=<找到的路径>
```

### 修复 D：重建 skill daemon bundle

当核心库刚被下载/重建，或 `dist/daemon.mjs` 不存在时执行：
```bash
cd ~/.claude/skills/Claude-to-IM-skill-main && npm run build
```

### 修复 E：设置开机自启动（任务计划程序）

当 `ClaudeToIMBridge` 计划任务不存在时执行。

运行：
```powershell
powershell -ExecutionPolicy Bypass -File "~/.claude/skills/Claude-to-IM-skill-main/scripts/register-autostart.ps1" install
```

这会注册一个"用户登录时"触发的计划任务，以当前用户身份在后台静默启动桥接 daemon。

验证：
```powershell
powershell -ExecutionPolicy Bypass -File "~/.claude/skills/Claude-to-IM-skill-main/scripts/register-autostart.ps1" status
```

如需移除自启动：
```powershell
powershell -ExecutionPolicy Bypass -File "~/.claude/skills/Claude-to-IM-skill-main/scripts/register-autostart.ps1" uninstall
```

---

## 修复完成后

所有修复完成后：

1. 如果桥接正在运行，重启它：
```powershell
powershell -ExecutionPolicy Bypass -File "~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1" stop
powershell -ExecutionPolicy Bypass -File "~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1" start
```

2. 告知用户修复完成，并提示：
   - 如果还没完成飞书后台配置（事件订阅、发布版本），需要先完成那些步骤
   - 可以用 `/claude-to-im status` 查看运行状态
   - 可以用 `/claude-to-im logs` 查看日志

---

## 注意事项

- 修复 supervisor-windows.ps1 时，使用 Edit 工具做精确替换，不要整体重写文件
- 下载核心库时，如果 GitHub API 返回 403（rate limit），提示用户稍后重试或手动克隆
- 所有修复都是幂等的：已经修复过的项目再次运行不会造成破坏
- 本 skill 只针对 Windows 环境；macOS/Linux 用户通常不需要这些修复
