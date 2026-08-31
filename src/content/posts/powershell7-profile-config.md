---
title: "PowerShell 7 终端配置：PSReadLine、补全模块与 UTF-8"
published: 2026-09-01
description: 'PowerShell 7 终端配置记录：列表式预测补全、PSCompletions 补全管理器、命令找不到时提示 winget 安装，外加 UTF-8 编码设置，改动都放在 $PROFILE 里。'
tags: [PowerShell, Windows, 终端]
draft: false
---

Windows 自带的 PowerShell 5.1 跑不动不少现代脚本，所以换成了 PowerShell 7。配置分三块：PSReadLine 交互、两个补全模块、UTF-8 编码，全写在 `$PROFILE` 一个文件里，换机器直接搬。

## 安装 PowerShell 7

```powershell
# winget
winget install --id Microsoft.PowerShell

# 或者 Scoop（versions bucket 里有）
scoop install versions/powershell-7
```

装完要开新的 `pwsh`，敲 `powershell` 还是 5.1。`$PSVersionTable.PSVersion` 显示 7.x 就对了，本文基于 7.6.5。

## 配置文件位置

```powershell
notepad $PROFILE
```

PowerShell 7 的默认路径是 `C:\Users\<用户名>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`，没有就创建。每次启动 `pwsh` 都会执行它。

## PSReadLine 交互设置

PSReadLine 随 PowerShell 7 内置，不用单独装。WinGet 模块靠它显示预测，要求 2.2.6+，版本旧了先更新：

```powershell
Update-Module PSReadLine
```

三行设置里，`PredictionViewStyle ListView` 把预测项从灰色内联文字改成列表，上下键选择，配合补全模块好用；`BellStyle None` 关掉 Tab 补全失败时的提示音；`EditMode Windows` 让键位按 Windows 习惯走，Home 到行首、Ctrl+z 撤销这些。最后一行把 Ctrl+z 显式绑成 Undo，Windows 模式下它本来就是撤销键，这里算声明一下，以后切到别的编辑模式也不丢。

```powershell
Set-PSReadLineOption -PredictionViewStyle ListView -BellStyle None -EditMode Windows
Set-PSReadLineKeyHandler -Key 'Ctrl+z' -Function Undo
```

## PSCompletions 补全管理器

[abgox](https://github.com/abgox) 的 [PSCompletions](https://github.com/abgox/PSCompletions)，用 Rust + Lua 实现。补全按命令添加，`psc add git` 加一个，内置补全库覆盖不少常用工具；补全菜单带中英文说明，展示顺序还会按命令历史调整。`psc update` 检查模块和补全库的更新。

```powershell
Install-Module PSCompletions          # PowerShellGet
# 或
Install-PSResource PSCompletions      # 7.4+ 默认的 PSResourceGet
# 或者 Scoop：先加 abyss bucket，再 scoop install abyss/abgox.PSCompletions
```

配置文件里导入，补全用的时候再加：

```powershell
Import-Module PSCompletions
psc add git    # 示例：给 git 加补全
```

重开终端，在 `git ` 后面按 Tab，就能看到参数、选项和说明。

## WinGet 命令找不到时的安装提示

官方 [Microsoft.WinGet.CommandNotFound](https://github.com/microsoft/winget-command-not-found)：敲了个没装的命令，它拿 winget 的包索引反查，预测区直接给安装命令。

```powershell
Install-Module Microsoft.WinGet.CommandNotFound
```

要求 PowerShell 7.4+、PSReadLine 2.2.6+。7.4 需要手动开两个实验特性；7.5 起 `PSCommandNotFoundSuggestion` 默认开启，7.6 装完就能用：

```powershell
Enable-ExperimentalFeature PSFeedbackProvider
Enable-ExperimentalFeature PSCommandNotFoundSuggestion
```

举个例子，本地没装 `ffmpeg` 时敲一下：

```
> ffmpeg
CommandNotFound: ffmpeg 不是可识别的 cmdlet、函数...
> winget install Gyan.FFmpeg
```

第二行就是预测补上的。

## UTF-8 编码

中文 Windows 控制台默认代码页是 GBK（936），外部程序输出中文、带中文名的文件经常乱码，`Export-Csv` 和 `Out-File` 默认也不是 UTF-8。四行设置：

```powershell
# 控制台输入/输出编码设为 UTF-8
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()

# PowerShell 管道和外部命令编码设为 UTF-8
$OutputEncoding = [System.Text.UTF8Encoding]::new()

# 所有 cmdlet（Export-Csv, Out-File 等）默认使用 UTF-8（无 BOM）
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
$env:LANG = 'zh_CN.UTF-8'
```

`[Console]::InputEncoding` / `[Console]::OutputEncoding` 管控制台本身，外部程序读写控制台的字符按 UTF-8 走；`$OutputEncoding` 管管道传给外部命令的文本；`$PSDefaultParameterValues['*:Encoding'] = 'utf8'` 让所有带 `-Encoding` 参数的 cmdlet 默认 UTF-8 无 BOM，`Export-Csv` 导出的文件 Excel 打开也不乱。`$env:LANG` 主要影响一部分跨平台原生工具取本地语言，Windows 上作用有限，顺手设的。

## 完整 $PROFILE

```powershell
# PSReadLine 交互体验设置
Set-PSReadLineOption -PredictionViewStyle ListView -BellStyle None -EditMode Windows
Set-PSReadLineKeyHandler -Key 'Ctrl+z' -Function Undo

# 导入模块（按需加载）
Import-Module PSCompletions
Import-Module -Name Microsoft.WinGet.CommandNotFound -ErrorAction SilentlyContinue

# 控制台输入/输出编码设为 UTF-8
[Console]::InputEncoding = [System.Text.UTF8Encoding]::new()
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new()

# PowerShell 管道和外部命令编码设为 UTF-8
$OutputEncoding = [System.Text.UTF8Encoding]::new()

# 所有 cmdlet（Export-Csv, Out-File 等）默认使用 UTF-8（无 BOM）
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
$env:LANG = 'zh_CN.UTF-8'
```

`Microsoft.WinGet.CommandNotFound` 那行带 `-ErrorAction SilentlyContinue`，winget 没装的时候静默跳过，不拖启动时间。哪个模块报错，用 `pwsh -NoProfile` 单独导入定位，再改 profile。

## 验证

重开 `pwsh`，或者 `. $PROFILE` 重载。检查这几个：

```powershell
$PSVersionTable.PSVersion    # 7.x
[Console]::OutputEncoding    # utf-8
Get-Module PSCompletions     # 已导入
```

然后命令行测试：`git ` 按 Tab 看补全；敲一个没装的命令（比如 `ffmpeg`），看预测区有没有 `winget install` 建议。都正常就结束了。

## 常见问题

- 预测不见：PSReadLine 版本低于 2.2.6，`Update-Module PSReadLine`。
- 补全空：PSCompletions 要先 `psc add <命令>`，导入不等于全有。
- 模块导入报错：`pwsh -NoProfile` 单测排查，从 profile 里先移掉报错的行。
