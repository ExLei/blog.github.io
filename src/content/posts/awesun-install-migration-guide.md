---
title: "向日葵远程控制（AweSun）装了什么:安装行为与自定义目录迁移指南"
published: 2026-09-01
description: "微软商店版 AweSun 实测:无 MSIX 的 Win32 安装形态;服务、自启、卸载注册、URL 协议、config.ini 五组引用;从 C 盘迁到自定义目录的完整步骤与通用脚本。"
tags: ["Windows"]
---

> 想弄清楚 AweSun 在机器上装了什么、又想把程序从 C 盘挪到别的盘,这篇适合你。下面的结论来自一台 Windows 11 机器的实测(微软商店安装的 AweSun 16.5.2.31214),不是从文档里抄的。

## 先说结论

微软商店里的向日葵,装完之后的形态是 **Win32 安装器的产物**,**没有 MSIX/Appx 包装**。商店只负责交付,最终落盘的跟官网下载安装器装出来的东西一样。

因为它确实是纯 Win32,整体搬去别的目录是可行的,前提是修好几处引用:服务、HKLM 自启键、卸载注册、URL 协议、程序内配置,再加两个开始菜单快捷方式。

迁移真正的难点不在"能不能搬",而在两处:① 服务与 HKLM 注册项需要管理员权限;② 命令行和注册表工具有几个坑,踩一个就中断。

## 1. 商店版为什么没有 MSIX

微软商店对传统桌面程序有一种"Win32 非包装分发":商店(或 App Installer)下载厂商的 EXE 安装器并运行,装完商店不留任何包。

这台机器的检查结果,四条全指向"无 MSIX":

| 检查 | 结果 |
|---|---|
| `Get-AppxPackage -Name '*Oray*'` / `'*AweSun*'` | 0 个包 |
| `HKCU\...\AppModel\Repository\Packages` 搜 `oray` | 无 |
| `HKLM\...\Appx\AppxAllUserStore` 搜 `oray` | 无 |
| 开始菜单入口 | 经典 `.lnk`(Appx 平铺不会产生 .lnk) |

商店"我的库"里的安装记录是账户历史,存在云端,本地没有文件可清。

## 2. 安装给系统留下了什么

### 2.1 安装目录

```
C:\Program Files\Oray\AweSun\
├── AweSun.exe            主程序(约 138 MB;服务、自启、卸载都调它)
├── install.bat           防火墙规则脚本(默认不自动执行)
├── config.ini            程序内配置(含 config_path)
├── md5.txt               安装包校验
├── RCHook.dll / node.dll
├── flutter\              Flutter 运行时 + 插件(Webview2Loader.dll、awesun-mcp-server.exe 等)
├── agent\AweSun.exe      桌面 Agent 副进程
├── awesun_guard\         看门狗进程
├── scad\ / itmprc\       组件库(itmprc 里有 rdp.exe、ssh.exe)
├── driver\               驱动源码包(Vhid64/VGC64/Idd64/Print64/OrayUSB*、DIFxAPI.x64.dll)
└── log\ / FileTransferFiles\
```

### 2.2 install.bat:防火墙

脚本接收程序名参数(如 `AweSun`),对两组程序各放行一次:主程序,以及 `agent\AweSun.exe`(规则名 `AweSun` / `AweSunDesktopAgent`)。每组在 public、domain、private 三个配置文件中放行 TCP/UDP 入站和出站。脚本里还留着老式 `netsh firewall` 分支(按系统版本判断走 `:old` 还是 `:new`)。

这台机器上,安装完成之后**没有任何防火墙规则**——install.bat 没有被运行过。远程连接靠 Windows 默认提示放行,想要显式规则,自己跑一次脚本即可。

### 2.3 注册与服务项(迁移时需要同步修改的地方)

| # | 位置 | 值 | 说明 |
|---|---|---|---|
| 1 | 服务 `AweSunService` | `binPath = "…\AweSun.exe" --mod=service`,AUTO_START,LocalSystem | 服务化运行模式 |
| 2 | `HKLM\SOFTWARE\...\CurrentVersion\Run` → `AweSun` | `"…\AweSun.exe" --cmd=autorun` | 登录自启,机器级 |
| 3 | `HKLM\SOFTWARE\WOW6432Node\...\Uninstall\Oray AweSun RemoteClient` | UninstallString / InstallLocation / DisplayIcon | 卸载入口(32 位安装器,故在 WOW6432Node) |
| 4 | 同上,`…\Oray Sunlogin RemoteClient` | 同样三个值 | 两个键并存,内容一致 |
| 5 | `HKLM\SOFTWARE\Classes\x-sl-awesun-auth` | `URL Protocol` 值 = exe 路径;`DefaultIcon` = `exe,1`;`shell\open\command` = `"exe" "%1"` | 协议注册,网页侧唤起程序用 |
| 6 | `HKLM\SOFTWARE\Oray\AweSun\AweSun` | `new_clientId` / `secret` / `machine_code` | **设备身份密钥**;删掉后设备在账号里会重新绑定 |
| 7 | `HKCU\Software\Oray\AweSun\AweSun` | 空键 | 占位 |
| 8 | 驱动服务 `OrayUSBVHCI`、`OrayVGC` | `\SystemRoot\System32\drivers\OrayUSBVHCI.sys` 等 | 内核驱动,路径与安装目录无关 |
| 9 | `HKCU\...\Shell\MuiCache` | `…\Oray\AweSun\flutter\AweSun.exe.FriendlyAppName` 等 | 友好名缓存,程序运行时会自动刷新,可清 |
| 10 | `HKCU\...\AppUserModelId\oray.sunlogin` | DisplayName / IconUri / CustomActivator 引用 | 与 App 标识相关;CustomActivator 的 CLSID 未注册 |

另外确认过没有:计划任务、App Paths 注册、其他 Oray 服务、防火墙规则。

### 2.4 驱动

驱动(USB 重定向、虚拟显卡、显示适配器)在安装时经 `DIFxAPI` 装入 DriverStore,同时复制到 `System32\drivers`,服务 `OrayUSBVHCI`、`OrayVGC` 的 ImagePath 就指向那里;PnP 层面另有三个驱动包(`orayusbvhci.inf`、`orayvgc.inf`、`orayidddriver.inf`)。这些跟安装目录没有路径关系,挪程序目录不影响驱动。

### 2.5 数据目录(跟程序目录分开的)

```
C:\ProgramData\Oray\AweSun\     机器级配置(sys_config.ini、fallback_data.ini、image\、log\)
C:\ProgramData\Oray\Webview2\   WebView2 运行时数据
%APPDATA%\Oray\AweSun\          用户级数据:confighive.hive、app_prefs.hive、orayhivestore.hive、
                                  accounts\、advertisement*.hive(广告缓存)、AweSunReport\、log\
```

登录态、设备身份、用户设置都在这些目录和注册表里,跟程序装在哪块盘无关——这是迁移能成立的基础。截图和文件传输的保存路径是用户自定义的,写在 `config.ini` 的 `screenshots_path` 等字段。

## 3. 迁移步骤(目录用变量代替)

```text
SRC  = C:\Program Files\Oray\AweSun          (原安装位置)
DEST = D:\Application\向日葵远程控制          (目标目录,按你的机器改)
```

### 方案 A:卸载重装,最省事

1. 开始菜单"卸载向日葵远程控制"(即 `AweSun.exe --mod=uninstall`)
2. 官网 oray.com 下载安装器
3. 安装时选目标目录——能不能选看安装器;不给选就走方案 B

### 方案 B:就地迁移(保配置、保设备绑定)

**前置:** ① 管理员终端;② 先退出程序(`sc stop AweSunService` + 任务管理器结束 AweSun.exe / awesun_guard.exe)。

**第 1 步:移动文件**

```cmd
robocopy "%SRC%" "%DEST%" /E /MOVE /R:1 /W:1
```

robocopy 的 `/MOVE` 复制成功后删源。普通用户可能删不掉 `C:\Program Files\Oray`(ACL 限制),剩下空壳目录用管理员补一刀:
`rmdir /s /q "C:\Program Files\Oray"`

**第 2 步:服务**

```cmd
sc config AweSunService binPath= "\"%DEST%\AweSun.exe\" --mod=service"
```

**第 3 步:登录自启**

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v AweSun /t REG_SZ /d "\"%DEST%\AweSun.exe\" --cmd=autorun" /f
```

**第 4 步:卸载注册,两个键各三处值,全部带 `/f`**

```cmd
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Oray AweSun RemoteClient" /v UninstallString /t REG_SZ /d "\"%DEST%\AweSun.exe\" --mod=uninstall" /f
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Oray AweSun RemoteClient" /v InstallLocation /t REG_SZ /d "%DEST%" /f
reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Oray AweSun RemoteClient" /v DisplayIcon /t REG_SZ /d "%DEST%\AweSun.exe" /f
:: Oray Sunlogin RemoteClient 同理,三行复制改键名
```

**第 5 步:URL 协议**

```cmd
reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth" /v "URL Protocol" /t REG_SZ /d "%DEST%\AweSun.exe" /f
reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth\DefaultIcon" /ve /t REG_SZ /d "%DEST%\AweSun.exe,1" /f
reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth\shell\open\command" /ve /t REG_SZ /d "\"%DEST%\AweSun.exe\" \"%%1\"" /f
```

**第 6 步:程序内配置**

改 `%DEST%\config.ini`:

```ini
config_path=D:\Application\向日葵远程控制\config.ini   ; 改成 %DEST%\config.ini
```

**第 7 步:快捷方式,删掉重建**

```powershell
Remove-Item 'C:\ProgramData\Microsoft\Windows\Start Menu\Programs\向日葵远程控制\向日葵远程控制.lnk' -Force
Remove-Item 'C:\ProgramData\Microsoft\Windows\Start Menu\Programs\向日葵远程控制\卸载向日葵远程控制.lnk' -Force
$w = New-Object -ComObject WScript.Shell
$s = $w.CreateShortcut('...\向日葵远程控制.lnk'); $s.TargetPath='%DEST%\AweSun.exe'; $s.Save()
$s2 = $w.CreateShortcut('...\卸载向日葵远程控制.lnk'); $s2.TargetPath='%DEST%\AweSun.exe'; $s2.Arguments='--mod=uninstall'; $s2.Save()
```

**第 8 步:启动**

```cmd
sc start AweSunService
```

### 可重复执行的脚本(通用版)

把下面这个 bat 放进 `%DEST%` 目录,用 `%~dp0` 自定位,不需要改任何路径,纯 ASCII:

```bat
@echo off
setlocal
set "NEW=%~dp0"
set "LOG=%NEW%migrate_log.txt"

sc stop AweSunService >nul 2>&1
if exist "C:\Program Files\Oray\AweSun" rmdir /s /q "C:\Program Files\Oray\AweSun"
sc config AweSunService binPath= "\"%NEW%AweSun.exe\" --mod=service"
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v AweSun /t REG_SZ /d "\"%NEW%AweSun.exe\" --cmd=autorun" /f

for %%K in ("Oray AweSun RemoteClient" "Oray Sunlogin RemoteClient") do (
  reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\%%~K" /v UninstallString /t REG_SZ /d "\"%NEW%AweSun.exe\" --mod=uninstall" /f
  reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\%%~K" /v InstallLocation /t REG_SZ /d "%NEW%" /f
  reg add "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\%%~K" /v DisplayIcon /t REG_SZ /d "%NEW%AweSun.exe" /f
)

reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth" /v "URL Protocol" /t REG_SZ /d "%NEW%AweSun.exe" /f
reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth\DefaultIcon" /ve /t REG_SZ /d "%NEW%AweSun.exe,1" /f
reg add "HKLM\SOFTWARE\Classes\x-sl-awesun-auth\shell\open\command" /ve /t REG_SZ /d "\"%NEW%AweSun.exe\" \"%%1\"" /f

sc start AweSunService
```

## 4. 常见问题

**迁移后要重新登录吗?** 不用。登录态在 `%APPDATA%\Oray\AweSun`(hive 数据)和注册表身份密钥里,与程序目录无关。

**迁移后防火墙怎么处理?** 这台机器安装时本来就没有规则(install.bat 没跑过)。需要显式放行就管理员运行 `"%DEST%\install.bat" AweSun`,规则名和程序路径按脚本所在目录重建。

**驱动要重装吗?** 不要。驱动在 DriverStore / System32,路径无关。

**商店版迁移后还能更新吗?** 它本来就没有 MSIX 包装,更新逻辑跟裸装版一样,与迁移不冲突;本机的 checkupdate 日志显示程序会做自检更新。
