# Nebula 分发避坑清单（打包 / 安装器必读）

> 面向**作者与打包者**。这里记录的是"开发机上一切正常、分发到别人电脑就翻车"的坑。
> 安装器（NSIS / Inno Setup / MSI / 便携版 `install.ps1`）必须逐条处理，否则用户拿到手就是
> **裸 `PS>` 提示符 + 豆腐块图标 + 偶发崩溃**。
>
> 判定标准：下面每一条都"自动化处理 + 有诊断手段 + 有回退"，才算分发就绪。

---

## 0. 快速自检（分发前在一台干净的 Windows 上跑一遍）

```powershell
# 1) 执行策略是否会挡脚本
Get-ExecutionPolicy -List

# 2) 字体是否装上（应能列出 Maple）
[System.Drawing.Text.InstalledFontCollection]::new().Families |
  Where-Object Name -match 'Maple'

# 3) powershell 能否在纯净 PATH 下被找到
where.exe powershell

# 4) TEMP 是否可写
"probe" | Out-File "$env:TEMP\nebula_probe.txt"; Remove-Item "$env:TEMP\nebula_probe.txt"
```

理想情况："纯净测试机"= 全新用户账户 / 新装的 Windows 虚拟机 / **域内受管理的机器**（企业电脑坑最多）。

---

## 1. 字体：MapleMono-NF-CN（不装 = 豆腐块）

**现象**：powerline 提示符的箭头 `❯`(U+276F)、git 分支图标、程序图标、AI 品牌图标全部显示成 □ 或缺字。
注意区分：

| 现象 | 根因 |
|------|------|
| 提示符是纯 `PS>`，无颜色 | **prompt 脚本没执行**（见 §2），不是字体问题 |
| 提示符有颜色和布局，但箭头/图标是 □ | **字体没装**（本节） |

**资源**：`assets/fonts/MapleMonoNormal-NF-CN-Regular.ttf`（SIL OFL 1.1，随包分发）。

### 安装器要做的：自动装字体（推荐 per-user，免管理员）

```powershell
# per-user 安装：不需要管理员权限，写当前用户字体目录 + HKCU 注册
$ttf  = 'MapleMonoNormal-NF-CN-Regular.ttf'
$src  = Join-Path $PSScriptRoot $ttf
$dir  = Join-Path $env:LOCALAPPDATA 'Microsoft\Windows\Fonts'
$dst  = Join-Path $dir $ttf
New-Item -ItemType Directory -Force -Path $dir | Out-Null
Copy-Item -LiteralPath $src -Destination $dst -Force

# 注册名必须是字体的“Full Name (TrueType)”，不是文件名。装一次后可用
#   (New-Object -ComObject Shell.Application).NameSpace($dir).ParseName($ttf).ExtendedProperty('System.Title')
# 确认真实字体名；Maple Mono Normal NF CN 的族名如下：
$name = 'Maple Mono Normal NF CN (TrueType)'
New-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows NT\CurrentVersion\Fonts' `
  -Name $name -Value $dst -PropertyType String -Force | Out-Null
```

要点/坑：
- **per-user 注册**（`HKCU` + `%LOCALAPPDATA%\...\Fonts`）Win10 1809+ 支持，免管理员，最适合便携版。
- **machine-wide** 需管理员：复制到 `C:\Windows\Fonts` 并写 `HKLM\...\Fonts`；MSI 用 `Font` 表更干净。
- 注册后**新开的进程**才会看到字体；已开的 Nebula 需重启。安装器装完提示"重启 Nebula"。
- 注册名写错（用了文件名而不是字体 Full Name）会导致"文件在、系统却认不出" —— 务必用实际族名。
- 便携版若不想碰注册表：Nebula 也可考虑**私有字体加载**（`AddFontResourceEx(..., FR_PRIVATE)` / DirectWrite 自定义字体集），把 ttf 跟随 exe 加载，零安装。**这是最省心的分发方式，建议纳入 roadmap。**

---

## 2. PowerShell 执行策略：分发后只剩裸 `PS>` 的头号元凶

**现象**：提示符是纯 `PS>`，没有颜色、没有 powerline —— prompt 集成脚本**根本没跑起来**。

**根因**：Nebula 启动 PowerShell 后需要加载集成脚本（`%TEMP%\nebula_prompt.ps1`：定义 `prompt` 函数、别名、PSReadLine、OSC 上报）。
在**域内 / 受管理 / 加固过的机器**上，组策略（GPO）会把 `MachinePolicy` 或 `UserPolicy` 的
ExecutionPolicy 强制成 `Restricted` / `AllSigned`。而策略优先级是：

```
MachinePolicy > UserPolicy > Process(-ExecutionPolicy 命令行参数) > CurrentUser > LocalMachine
```

也就是说，命令行的 `-ExecutionPolicy Bypass` **打不过 GPO** —— 于是 dot-source 一个未签名的 `.ps1`
被拒，`prompt` 函数没定义，PowerShell 回退默认提示符 = `PS>`。

### 代码侧已修（主保险，已在 `nebula_terminal/src/tty/windows/mod.rs`）

不再 `. '脚本.ps1'`（执行**脚本文件**，受策略管），改为：

```powershell
Get-Content -LiteralPath '<临时脚本>' -Raw | Invoke-Expression
```

`Invoke-Expression` 执行的是**内存里的命令字符串**，不是"运行脚本文件"，**ExecutionPolicy 不拦 IEX**。
这样即便 GPO 强制 Restricted，powerline 依旧生效。**这是覆盖面最广的修复，无需用户配合。**

### 安装器侧建议（双保险 + 诊断）

- 装完提示用户可自查：`Get-ExecutionPolicy -List`。若 `MachinePolicy`/`UserPolicy` 非 `Undefined`，说明被 GPO 接管。
- 无 GPO 的普通机器，安装器可顺手：
  ```powershell
  Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force  # 打不过 GPO，但对无策略机有益
  ```
- **不要**建议用户全局 `Set-ExecutionPolicy Bypass`（削弱系统安全，且仍打不过 GPO）。
- 残留兜底：`ConstrainedLanguage` 语言模式（WDAC/AppLocker）下，IEX 也会因禁用 .NET 调用而失败。
  这属于极端加固环境，暂不保证 powerline，但 shell 本身可用 —— 文档注明即可。

---

## 3. 新建 tab / 分屏崩溃：`Failed to spawn pane: 目录名称无效 (os error 267)`

**现象**：开新标签或分屏时红条报错 `os error 267 (ERROR_DIRECTORY)`。

**根因**：新 pane 继承"当前 pane 的工作目录"（shell 通过 OSC 标题上报的 cwd）。若该目录：
- 已被删除 / 在已卸载的驱动器上；
- 是 PowerShell 的**非文件系统 PSDrive**（`cd Cert:\`、`cd HKLM:\`、`cd Env:\` 后 `$PWD` 不是真实目录）；
- 是 CreateProcessW 不接受的路径（部分 UNC / 虚拟路径），

`CreateProcessW` 的 `lpCurrentDirectory` 就会失败，整个 pane 起不来。

**代码侧已修**（`nebula_app/src/window_context.rs`）：
- `create_pane`（所有 pane 的唯一入口）spawn 前校验 `working_directory.is_dir()`，无效则清空回退到进程默认目录；
- `focused_cwd()` 复用 `session::valid_dir()` 做 `is_dir()` 校验，与会话恢复路径行为一致。

打包者无需额外动作，但要知道：**这是"某些设备"偶发崩溃，别当成随机 bug。** 复现方法：`cd HKLM:\` 后按新建标签。

---

## 4. `ms-gamingoverlay` 弹窗（部分设备）

**现象**：弹出「无法打开此 "ms-gamingoverlay" 链接 / 你的设备需要一个新应用」。

**根因**：Windows 的 Xbox Game Bar 协议被触发（常见于 `Win+G`，或某些按键/窗口消息冒泡到系统 Game Bar 处理），
而该设备**卸载了 Xbox Game Bar**（精简版系统、LTSC、手动移除），于是系统找不到 `ms-gamingoverlay:` 协议处理器就报错。

**缓解手段（分发/文档二选一或都给）**：
- 应用侧（首选，待代码验证）：拦截 / 不向系统透传会触发 Game Bar 的组合键；确保窗口不会把 `Win+G` 之类冒泡。
- 系统侧（安装器可选，或写进 FAQ）：关闭 Game Bar 弹层，
  ```powershell
  # 每用户关闭 Game Bar（不影响其它功能）
  New-Item -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\GameDVR' -Force | Out-Null
  Set-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\GameDVR' AppCaptureEnabled 0
  Set-ItemProperty 'HKCU:\System\GameConfigStore' GameDVR_Enabled 0
  ```
- **不要**默认帮用户改系统级 Game Bar 设置而不告知；写进 FAQ 让用户自行决定更稳妥。

> 状态：待在代码侧确认 Nebula 是否真的透传了触发键。若确认，应在按键处理层吞掉，从根上消除。

---

## 5. 临时目录与杀软

- Nebula 每次 spawn 会把集成脚本写到 `%TEMP%`（`nebula_prompt.ps1` / `nebula_bashrc`），内容不变则复用、不重复写。
- **坑**：`%TEMP%` 不可写（磁盘满 / 权限 / 重定向到无效路径）→ 集成脚本写不出 → 回退无 prompt。
- **坑**：杀软 / EDR 对"落地 `.ps1` 后立刻被执行"敏感，可能拦截或延迟（首个 prompt 卡顿）。
  - IEX 方案（§2）已不再"运行脚本文件"，能降低部分误报；
  - 分发时建议对 Nebula 主程序做**代码签名**，显著减少 SmartScreen / AV 拦截。
- 服务 / 计划任务 / SYSTEM 上下文运行时 `%TEMP%` 可能指向 `C:\Windows\Temp`，注意权限。

---

## 6. `powershell` 的定位依赖 PATH

- 代码用裸名 `powershell` 启动（靠 PATH 找 `powershell.exe`）。
- 正常机器 `C:\Windows\System32\WindowsPowerShell\v1.0` 在系统 PATH 中，没问题。
- **坑**：被裁剪过 PATH 的机器 / 某些精简系统可能找不到 → shell 起不来。
- 注意：仅安装 PowerShell 7（`pwsh.exe`）而**移除了 Windows PowerShell** 的机器上，`powershell` 不存在。
  若要支持，需增加 `pwsh` 回退探测。

---

## 7. Server / 保活进程（务必在 README 向用户说明）

- Nebula 存在一个 **server / 保活进程**：它的作用是**保活**（窗口全部关闭后仍维持会话 / 快速重开 / 后台任务不被杀）。
- **必须在 README 与 FAQ 明确告知用户**：这是设计使然，不是"卸载不干净"或"后台偷跑"。
  说明：进程名、为什么存在（保活）、如何彻底退出（完全退出而非关窗）、是否随开机自启、如何关闭该行为。
- 否则用户在任务管理器看到"关了窗口还有进程"会误判为流氓软件。

---

## 分发就绪检查表（Definition of Done）

- [ ] 安装器**自动安装字体**（per-user 免管理员），装完提示重启 Nebula
- [ ] powerline 在**域内受管理机器**上仍生效（§2 IEX 已保证；实测一台 GPO 机确认）
- [ ] `Get-ExecutionPolicy -List` 写进 FAQ 作为自查手段
- [ ] os error 267 已修（`is_dir` 校验回退）—— 冒烟：`cd HKLM:\` 后新建标签不崩
- [ ] `ms-gamingoverlay` 已在代码侧确认/缓解，或写进 FAQ
- [ ] 主程序**代码签名**，降低 SmartScreen / AV 拦截
- [ ] README 说明 **server 保活进程**的存在与退出方式
- [ ] 便携版在**纯净 PATH / 无网络 / 非管理员**账户下自检通过
