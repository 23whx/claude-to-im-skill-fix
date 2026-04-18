---
name: claude-to-im-skill-fix
description: 修复 Claude-to-IM-skill 在 Windows 上的已知安装和兼容性问题。当用户说 /claude-to-im-skill-fix、"帮我修复飞书桥接"、"Claude-to-IM 跑不起来"、"fix claude to im" 等时触发。
---

# Claude-to-IM-skill-fix

你是 Claude-to-IM-skill 的 Windows 修复助手，负责诊断并修复已知问题。

## 重要：工具使用规则

- 运行命令：使用 **Bash 工具**（不要直接输出命令让用户执行，除非明确说明）
- 读取文件：使用 **Read 工具**
- 修改文件：使用 **Edit 工具**（精确替换，不要整体重写）
- 创建文件：使用 **Write 工具**
- 询问用户：使用 **AskUserQuestion 工具**（如果可用）；如果不可用，直接用文字问用户

## 路径说明

本文档中的路径变量含义：
- `TARGET_DIR` = `C:\Users\<用户名>\.claude\skills\Claude-to-IM-skill-main`
- `CORE_DIR` = `C:\Users\<用户名>\.claude\skills\Claude-to-IM`
- `SKILL_DIR` = `C:\Users\<用户名>\.claude\skills\claude-to-im-skill-fix`
- `CTI_HOME` = `C:\Users\<用户名>\.claude-to-im`

在 Bash 工具中，`~` 等同于上面的 `C:\Users\<用户名>`，可以直接使用。

---

## 第一步：运行诊断

依次执行以下每一项检查，记录每项结果（正常 / 有问题 / 未安装）。

### 检查 1：TARGET_DIR 是否存在

用 Bash 工具运行：
```bash
test -d ~/.claude/skills/Claude-to-IM-skill-main && echo "EXISTS" || echo "NOT_FOUND"
```

- 输出 `EXISTS`：✅ 正常
- 输出 `NOT_FOUND`：❌ 问题——需要先安装 skill，告知用户运行：
  ```
  npx skills add op7418/Claude-to-IM-skill
  ```
  然后停止，等用户安装完再继续。

### 检查 2：CORE_DIR 是否存在且完整

用 Bash 工具运行：
```bash
test -f ~/.claude/skills/Claude-to-IM/src/lib/bridge/context.ts && echo "SRC_OK" || echo "SRC_MISSING"
test -f ~/.claude/skills/Claude-to-IM/dist/lib/bridge/context.js && echo "DIST_OK" || echo "DIST_MISSING"
```

- 两行都是 OK：✅ 正常
- `SRC_MISSING`：❌ 问题——需要执行修复 A（下载核心库）
- `SRC_OK` 但 `DIST_MISSING`：❌ 问题——需要执行修复 D（重新构建）

### 检查 3：supervisor-windows.ps1 的 PS5.1 兼容性

用 Bash 工具运行以下四条命令，每条单独运行：

**检查 3a — 三参数 Join-Path（PS5.1 不支持）：**
```bash
grep -P "Join-Path\s+\S+\s+'[^']+'\s+'[^']+'" ~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1 && echo "BUG_FOUND" || echo "OK"
```
- 输出 `BUG_FOUND`：❌ 需要执行修复 B1
- 输出 `OK`：✅ 正常

**检查 3b — stdout 和 stderr 重定向到同一文件：**
```bash
grep -c "RedirectStandardError \$LogFile" ~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1
```
- 输出 `1` 或更大：❌ 需要执行修复 B2
- 输出 `0`：✅ 正常

**检查 3c — $pid 只读变量冲突：**
```bash
grep -iP "\\\$pid\b" ~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1 | grep -v "ProcessId\|bridgePid\|#" && echo "BUG_FOUND" || echo "OK"
```
- 输出 `BUG_FOUND`：❌ 需要执行修复 B3
- 输出 `OK`：✅ 正常

**检查 3d — config.env 环境变量注入：**
```bash
grep -c "SetEnvironmentVariable" ~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1
```
- 输出 `0`：❌ 需要执行修复 B4
- 输出 `1` 或更大：✅ 正常

### 检查 4：config.env 配置文件

用 Bash 工具运行：
```bash
test -f ~/.claude-to-im/config.env && echo "EXISTS" || echo "NOT_FOUND"
```

- 输出 `NOT_FOUND`：❌ 问题——告知用户需要先运行 `/claude-to-im setup` 完成配置，然后停止。
- 输出 `EXISTS`：继续检查 Git Bash 路径：

```bash
grep "CLAUDE_CODE_GIT_BASH_PATH" ~/.claude-to-im/config.env && echo "SET" || echo "NOT_SET"
```

- 输出 `NOT_SET`：❌ 需要执行修复 C
- 输出包含路径：✅ 正常

### 检查 5：开机自启动任务

用 Bash 工具运行：
```bash
powershell -NonInteractive -Command "if (Get-ScheduledTask -TaskName 'ClaudeToIMBridge' -ErrorAction SilentlyContinue) { 'FOUND' } else { 'NOT_FOUND' }"
```

- 输出 `FOUND`：✅ 已设置自启动
- 输出 `NOT_FOUND`：⚠️ 未设置（可选修复 E，询问用户是否需要）

### 检查 6：Node.js 版本

用 Bash 工具运行：
```bash
node --version
```

- 输出 `v20.x.x` 或更高：✅ 正常
- 输出低于 v20，或命令不存在：❌ 告知用户需要安装 Node.js >= 20，然后停止。

### 检查 7：Claude CLI

用 Bash 工具运行：
```bash
claude --version 2>&1 | head -1
```

- 输出包含版本号（如 `2.x.x`）：✅ 正常
- 命令不存在：❌ 告知用户需要安装 Claude Code CLI，然后停止。

---

## 第二步：汇报诊断结果

将所有检查结果整理成表格展示给用户：

| 检查项 | 状态 | 说明 |
|--------|------|------|
| TARGET_DIR | ✅/❌ | ... |
| CORE_DIR | ✅/❌ | ... |
| PS5.1 兼容性 | ✅/❌ | ... |
| config.env | ✅/❌ | ... |
| 开机自启动 | ✅/⚠️ | ... |
| Node.js | ✅/❌ | ... |
| Claude CLI | ✅/❌ | ... |

如果所有项目都是 ✅，告知用户"环境正常，无需修复"，结束。

---

## 第三步：询问用户确认

列出所有需要修复的项目（❌ 和用户选择修复的 ⚠️）。

如果 AskUserQuestion 工具可用，用它询问：
> "发现以上问题，是否现在执行修复？"
> 选项：是 / 否

如果 AskUserQuestion 工具不可用，直接用文字问用户，等待回复后再继续。

用户确认后，进入第四步。

---

## 第四步：执行修复

按照下面的修复方法，依次处理每个需要修复的项目。每完成一项，告知用户进度。

---

## 修复方法

### 修复 A：下载并构建核心库 Claude-to-IM

**适用场景**：检查 2 发现 `SRC_MISSING`。

**步骤 A1 — 创建目录结构**

用 Bash 工具运行：
```bash
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/adapters
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/markdown
mkdir -p ~/.claude/skills/Claude-to-IM/src/lib/bridge/security
```

**步骤 A2 — 逐文件下载源码**

直接 `git clone` 在某些网络环境会被重置，改用 GitHub API（更稳定）。

对下面列表中的每个文件路径，用 Bash 工具运行以下 PowerShell 命令（将 `{FILE_PATH}` 替换为实际路径，将 `{DEST_DIR}` 替换为目标目录的完整路径）：

```bash
powershell -NonInteractive -Command "
  \$path = '{FILE_PATH}'
  \$dest = '{DEST_DIR}'
  \$resp = Invoke-RestMethod \"https://api.github.com/repos/op7418/Claude-to-IM/contents/\$path\"
  \$bytes = [System.Convert]::FromBase64String((\$resp.content -replace '\n',''))
  \$outPath = Join-Path \$dest (\$path -replace '/','\')
  \$outDir = Split-Path \$outPath
  if (-not (Test-Path \$outDir)) { New-Item -ItemType Directory -Path \$outDir -Force | Out-Null }
  [System.IO.File]::WriteAllBytes(\$outPath, \$bytes)
  Write-Output \"OK: \$path\"
"
```

需要下载的文件列表（逐个处理）：
```
package.json
tsconfig.json
tsconfig.build.json
src/lib/bridge/types.ts
src/lib/bridge/host.ts
src/lib/bridge/context.ts
src/lib/bridge/channel-adapter.ts
src/lib/bridge/channel-router.ts
src/lib/bridge/bridge-manager.ts
src/lib/bridge/conversation-engine.ts
src/lib/bridge/delivery-layer.ts
src/lib/bridge/permission-broker.ts
src/lib/bridge/adapters/index.ts
src/lib/bridge/adapters/feishu-adapter.ts
src/lib/bridge/adapters/telegram-adapter.ts
src/lib/bridge/adapters/telegram-media.ts
src/lib/bridge/adapters/telegram-utils.ts
src/lib/bridge/adapters/discord-adapter.ts
src/lib/bridge/adapters/qq-adapter.ts
src/lib/bridge/adapters/qq-api.ts
src/lib/bridge/markdown/ir.ts
src/lib/bridge/markdown/render.ts
src/lib/bridge/markdown/feishu.ts
src/lib/bridge/markdown/telegram.ts
src/lib/bridge/markdown/discord.ts
src/lib/bridge/security/rate-limiter.ts
src/lib/bridge/security/validators.ts
```

如果某个文件下载时 GitHub API 返回 403，说明触发了速率限制，告知用户稍等 1 分钟后重试。

**步骤 A3 — 安装依赖并构建**

用 Bash 工具运行：
```bash
cd ~/.claude/skills/Claude-to-IM && npm install && npm run build
```

等待完成。如果报错，将错误信息展示给用户。

**步骤 A4 — 刷新 skill 依赖**

用 Bash 工具运行：
```bash
cd ~/.claude/skills/Claude-to-IM-skill-main && npm install
```

---

### 修复 B1：修复三参数 Join-Path（PS5.1 不支持）

**适用场景**：检查 3a 发现 `BUG_FOUND`。

先用 Read 工具读取文件：
```
~/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1
```

然后用 Edit 工具，将：
```powershell
$LogFile    = Join-Path $CtiHome 'logs' 'bridge.log'
$DaemonMjs  = Join-Path $SkillDir 'dist' 'daemon.mjs'
```
替换为：
```powershell
$LogFile    = Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'
$DaemonMjs  = Join-Path (Join-Path $SkillDir 'dist') 'daemon.mjs'
```

---

### 修复 B2：修复 stdout/stderr 重定向到同一文件

**适用场景**：检查 3b 发现问题。

先用 Read 工具读取文件（如果修复 B1 刚读过，可跳过）。

用 Edit 工具，在路径定义区域（`$LogFile` 那一行附近）添加 `$StderrFile` 变量。找到：
```powershell
$LogFile    = Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'
```
替换为：
```powershell
$LogFile    = Join-Path (Join-Path $CtiHome 'logs') 'bridge.log'
$StderrFile = Join-Path (Join-Path $CtiHome 'logs') 'bridge-stderr.log'
```

再用 Edit 工具，找到：
```powershell
        -RedirectStandardError $LogFile `
```
替换为：
```powershell
        -RedirectStandardError $StderrFile `
```

---

### 修复 B3：修复 $pid 只读变量冲突

**适用场景**：检查 3c 发现问题。

先用 Read 工具读取文件（如果已读过，可跳过）。

用 Edit 工具做以下替换（每次替换后确认成功再做下一个）：

1. 将函数参数名替换：
   - 找：`param([string]$Pid)`
   - 换：`param([string]$ProcessId)`

2. 将函数体内的变量名替换：
   - 找：`if (-not $Pid) { return $false }`
   - 换：`if (-not $ProcessId) { return $false }`

3. 将函数体内的变量名替换：
   - 找：`try { $null = Get-Process -Id ([int]$Pid) -ErrorAction Stop; return $true }`
   - 换：`try { $null = Get-Process -Id ([int]$ProcessId) -ErrorAction Stop; return $true }`

4. 将 start 命令块中的局部变量替换：
   - 找：`$pid = Start-Fallback`
   - 换：`$bridgePid = Start-Fallback`

5. 将 stop 命令块中的局部变量替换：
   - 找：`$bridgePid = Read-Pid` 如果已存在则跳过；否则找 `$pid = Read-Pid`（在 stop 块中）替换为 `$bridgePid = Read-Pid`

6. 将 status 命令块中的局部变量替换（同上逻辑）

**注意**：每次 Edit 替换前，确认 old_string 在文件中确实存在且唯一，否则加上更多上下文使其唯一。

---

### 修复 B4：添加 config.env 环境变量注入

**适用场景**：检查 3d 发现问题（`SetEnvironmentVariable` 出现次数为 0）。

先用 Read 工具读取文件（如果已读过，可跳过）。

**步骤 1**：在路径定义区域添加 `$ConfigFile` 变量。用 Edit 工具，找到：
```powershell
$ServiceName = 'ClaudeToIMBridge'
```
替换为：
```powershell
$ServiceName = 'ClaudeToIMBridge'
$ConfigFile  = Join-Path $CtiHome 'config.env'
```

**步骤 2**：在 `start` 命令的 `else` 分支中，找到以下内容（这是 fallback 启动路径的开头）：
```powershell
            Write-Host "Starting bridge (background process)..."
            $bridgePid = Start-Fallback
```
替换为：
```powershell
            # Load config.env vars into the current session so the node process inherits them
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
            # Remove CLAUDECODE so the daemon doesn't think it's running inside Claude Code
            [System.Environment]::SetEnvironmentVariable('CLAUDECODE', $null, 'Process')

            Write-Host "Starting bridge (background process)..."
            $bridgePid = Start-Fallback
```

---

### 修复 C：设置 CLAUDE_CODE_GIT_BASH_PATH

**适用场景**：检查 4 发现 `config.env` 中没有 `CLAUDE_CODE_GIT_BASH_PATH`。

**步骤 1**：用 Bash 工具查找 bash.exe 位置：
```bash
where bash 2>/dev/null || powershell -NonInteractive -Command "Get-Command bash -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source"
```

记录输出的路径（例如 `D:\devsoft\Git\bin\bash.exe`）。

如果找不到，告知用户需要安装 Git for Windows，然后停止。

**步骤 2**：用 Bash 工具将路径追加到 config.env：
```bash
echo "CLAUDE_CODE_GIT_BASH_PATH=<上一步找到的路径>" >> ~/.claude-to-im/config.env
```

将 `<上一步找到的路径>` 替换为实际路径，注意路径中的反斜杠在 bash 中需要用正斜杠或双反斜杠。

---

### 修复 D：重建 skill daemon bundle

**适用场景**：检查 2 发现 `DIST_MISSING`，或修复 A 完成后需要重建 skill 层。

用 Bash 工具运行：
```bash
cd ~/.claude/skills/Claude-to-IM-skill-main && npm run build
```

等待完成。如果报错，将错误信息展示给用户。

---

### 修复 E：设置开机自启动（任务计划程序）

**适用场景**：检查 5 发现 `NOT_FOUND`，且用户确认需要设置。

用 Bash 工具运行：
```bash
powershell -NonInteractive -ExecutionPolicy Bypass -File "$HOME/.claude/skills/Claude-to-IM-skill-main/scripts/register-autostart.ps1" install
```

验证是否成功：
```bash
powershell -NonInteractive -Command "if (Get-ScheduledTask -TaskName 'ClaudeToIMBridge' -ErrorAction SilentlyContinue) { 'SUCCESS' } else { 'FAILED' }"
```

- 输出 `SUCCESS`：✅ 自启动设置成功
- 输出 `FAILED`：❌ 告知用户安装失败，建议手动运行上面的 ps1 脚本

---

## 第五步：修复完成后

所有修复执行完毕后：

1. 如果执行了修复 B（修改了 supervisor-windows.ps1），需要重启桥接。用 Bash 工具运行：
```bash
powershell -NonInteractive -ExecutionPolicy Bypass -File "$HOME/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1" stop
sleep 2
powershell -NonInteractive -ExecutionPolicy Bypass -File "$HOME/.claude/skills/Claude-to-IM-skill-main/scripts/supervisor-windows.ps1" start
```

2. 告知用户修复完成，并提示：
   - 如果还没完成飞书后台配置（事件订阅、发布版本），需要先完成那些步骤
   - 可以用 `/claude-to-im status` 查看运行状态
   - 可以用 `/claude-to-im logs` 查看日志

---

## 注意事项

- 修复 B 系列时，使用 Edit 工具做精确替换，不要整体重写文件
- 每次 Edit 替换前，确认 old_string 在文件中唯一；如果不唯一，加上更多上下文
- 下载核心库时，如果 GitHub API 返回 403，提示用户稍等 1 分钟后重试
- 所有修复都是幂等的：已经修复过的项目再次运行不会造成破坏
- 本 skill 只针对 Windows 环境；macOS/Linux 用户通常不需要这些修复
