---
name: dde-log-analyzer
description: >
  分析 DDE（深度桌面环境）系统日志，帮助定位桌面组件故障。
  当用户提到 DDE 桌面问题、桌面崩溃、登录异常、dock 栏问题、控制中心问题、
  应用启动失败、系统更新问题、剪贴板问题、外观/主题问题、D-Bus 服务异常、
  或任何 deepin/DDE 相关的系统排查需求时使用此技能。
  本技能帮助 AI 知道去哪里找日志、如何过滤、如何重启/调试各组件、
  如何查看和修改 DConfig 配置、以及各组件的作用。
  适用场景包括但不限于：桌面黑屏、dock 栏消失、控制中心打不开、
  应用启动器异常、锁屏问题、通知问题、系统更新失败、
  剪贴板不工作、主题切换失败、D-Bus 服务报错等。
---

# DDE 日志分析指南

当用户报告 DDE 桌面问题时，使用本技能作为参考，知道去哪里找日志、如何过滤、各组件的作用，以及如何快速重启和调试。

## 重要前提

**优先保留问题现场，不要轻易执行重启操作。** 这个 skill 用于公司内部排查共性问题，重启会丢失现场信息。排查时应：

1. 先收集日志和状态信息，再考虑重启
2. 如果需要重启验证，先告知用户可能丢失哪些现场信息，让用户决定
3. 对于可复现的问题，保留现场更有价值——方便分析共性原因
4. 只有在用户明确要求快速恢复、或问题无法继续排查时，才建议重启

## 日志查看总原则

DDE 组件的主要日志来源是 **systemd journal**（用户级），少数组件有文件日志。

### 常用 journal 查询命令

```bash
# 查看所有 DDE 相关日志（最近）
journalctl --user -b 0 --no-pager | grep -i "dde\|deepin"

# 按某个服务过滤
journalctl --user -u <service-name> -b 0 --no-pager

# 按时间范围
journalctl --user --since "10 min ago" --no-pager | grep -i "dde"

# 只看错误级别
journalctl --user -b 0 -p err --no-pager | grep -i "dde\|deepin"

# 按二进制路径查看日志（适用于独立二进制工具）
journalctl --user --no-pager -b 0 /usr/bin/<binary-name>
```

### 文件日志位置

大多数组件不产生文件日志。少数例外：
- `~/.cache/deepin/<app-id>/<app-id>.log` — 个别组件（如 tray-loader）

### 开启 Debug 日志

**Go 项目**（dde-daemon, dde-api）：设置环境变量 `DDE_DEBUG_LEVEL=debug`

**Qt 项目**：通过 Qt 日志环境变量，如 `QT_LOGGING_RULES="*.debug=true"`

**Wayland 调试**（tray-loader 等运行在 dde-shell 合成器中的组件）：设置 `WAYLAND_DEBUG=1` 可以看到 Wayland 协议层的通信日志

**控制中心 GUI**：控制中心 → 开发者选项 → 打开调试日志开关

---

## dde-shell 重启指南

dde-shell 是 DDE 的核心组件，它只管理两个 systemd 服务：

### 重启整个 DDE 桌面环境
```bash
systemctl --user restart dde-shell@DDE.service
```
这会重新加载所有 dde-shell 插件（dock、桌面、通知、OSD 等），适用于整个桌面卡死或异常的情况。

### 只重启桌面（不重启 dock 等其他插件）
```bash
systemctl --user restart dde-shell-plugin@org.deepin.ds.desktop.service
```
适用于桌面背景异常、右键菜单不工作等桌面本身的问题，不影响 dock 和其他组件。

### dde-shell 与 Wayland 合成器

dde-shell 内置了 Wayland 合成器，**tray-loader 等插件组件运行在这个合成器中**。这意味着：

- tray-loader 启动时需要加 `--platform wayland` 参数来跑在 dde-shell 的合成器里面
- 遇到显示/渲染问题时，可以通过 `WAYLAND_DEBUG=1` 环境变量来看 Wayland 协议层的日志
- 如果 dde-shell 的合成器出问题，所有运行在其中的插件都会受影响

---

## DDE 组件速查表

### 核心会话管理

| 组件 | 作用 | 服务名 | 日志 |
|------|------|--------|------|
| **dde-session** | 会话管理器，启动/管理所有 DDE 组件 | `dde-session-manager.service` | journalctl -f /usr/bin/dde-session |
| **dde-shell** | 桌面 shell，加载 dock/面板/通知等插件 | `dde-shell@DDE.service`, `dde-shell-plugin@org.deepin.ds.desktop.service` | journal (user), QLoggingCategory `dsLog` |

### 桌面 UI 组件

| 组件 | 作用 | 日志 |
|------|------|------|
| **dde-session-shell** | 锁屏和登录界面 (dde-lock, lightdm-deepin-greeter) | journal (user) |
| **dde-control-center** | 控制中心/系统设置 | journal (user) |
| **dde-launchpad** | 应用启动器/开始菜单 | journal (user) |
| **dde-clipboard** | 剪贴板管理 | journal (user) |

### 后台服务/守护进程

| 组件 | 作用 | 服务名 | 日志 |
|------|------|--------|------|
| **dde-daemon** | 后台设置守护进程 (Go)，管理显示、账户、蓝牙、电源等 | 常驻: `dde-session-daemon`, `dde-system-daemon` | journal (system + user) |
| **dde-appearance** | 外观/主题设置守护进程 | `org.deepin.dde.Appearance1.service`, `dde-fakewm.service` | journal (user) |
| **dde-application-manager** | 应用管理（desktop entry 解析、启动、自启） | `org.desktopspec.ApplicationManager1.service`, `dde-autostart.service` | journal (user) |
| **deepin-service-manager** | D-Bus 服务管理器，管理插件式 D-Bus 服务（含 dde-appearance 的个性化服务和 dde-services 的服务） | — | `sudo journalctl -f /usr/bin/deepin-service-manager` |
| **dde-dconfig-daemon** | DConfig 配置管理守护进程 | `dde-dconfig-daemon.service` | journal (system) |

### 其他组件

| 组件 | 作用 | 日志 |
|------|------|------|
| **dde-tray-loader** | Dock 栏托盘插件加载器（运行在 dde-shell 的 Wayland 合成器中） | 文件日志: `~/.cache/deepin/org.deepin.dde.tray-loader/org.deepin.dde.tray-loader.log`; journal (user) |
| **dde-polkit-agent** | 权限认证代理 | journal (user) |
| **deepin-update-ui** | 系统更新 UI（见下方更新日志说明） | 见下方 |
| **dde-network-core** | 网络插件库（控制中心和 dock 都用） | 在各自框架中查看 |
| **dde-api** | Go 系统 API (声音、设备、语言等) | journal (user) |
| **dde-services** | 插件式 D-Bus 服务框架 | journal (user) |
| **dde-app-services** | DConfig 配置中心 | journal (user) |

---

## 系统更新日志

deepin-update-ui 涉及的更新日志分三种情况：

### 1. 控制中心的更新模块日志
在控制中心的 GUI 界面中可以直接查看更新状态和日志。这是最方便的方式。

### 2. 模态更新（全屏覆盖不允许用户操作界面）
模态更新的二进制日志需要 root 权限查看：
```bash
# 查看已有日志
sudo journalctl -b 0 /usr/bin/dde-update --no-pager
# 实时跟踪（会阻塞终端，用 Ctrl+C 退出）
sudo journalctl -f /usr/bin/dde-update
```

### 3. 更新后端（lastore-daemon）
```bash
# 查看已有日志
sudo journalctl -b 0 /usr/libexec/lastore-daemon/lastore-daemon --no-pager
# 实时跟踪（会阻塞终端，用 Ctrl+C 退出）
sudo journalctl -f /usr/libexec/lastore-daemon/lastore-daemon
```
需要 root 权限。

---

## 网络插件日志

dde-network-core 被两个框架使用，日志在各自的框架中查看：

- **控制中心的网络模块**：在控制中心日志中查看（`journalctl --user | grep -i "control-center"`）
- **dock 栏的网络插件**：在 dde-tray-loader 的日志中查看（`~/.cache/deepin/org.deepin.dde.tray-loader/org.deepin.dde.tray-loader.log`）

---

## DConfig 配置管理

DConfig 是 DDE 的配置管理系统，由 `dde-app-services` 项目提供。排查问题时经常需要查看或修改某个组件的配置。

### dde-dconfig 命令行工具

```bash
# 查看所有可配置的应用列表
dde-dconfig list

# 获取某个配置项的值
dde-dconfig get -a <appid> -r <resource> -k <key>

# 设置某个配置项的值
dde-dconfig set -a <appid> -r <resource> -k <key> -v <value>

# 重置某个配置项（恢复默认值）
dde-dconfig reset -a <appid> -r <resource> -k <key>

# 监听某个配置项的变更
dde-dconfig watch -a <appid> -r <resource> -k <key>

# 打开 GUI 编辑器
dde-dconfig gui

# 查看帮助
dde-dconfig --help
```

**常用示例**：
```bash
# 查看 XSettings 的 DPI 设置
dde-dconfig get -a org.deepin.dde.daemon -r org.deepin.XSettings -k xft-dpi

# 查看控制中心的某个开关
dde-dconfig get -a org.deepin.dde.control-center -k <key-name>
```

### DConfig 配置文件位置

- **Schema 定义文件**（只读）：`/usr/share/dsg/configs/<appid>/` — 包含配置的定义和默认值
- **用户缓存文件**（可读写，需要 root）：`/var/lib/dde-dconfig-daemon/.config/<uid>/<appid>/<appid>.json`
  - `<uid>` 是用户 ID（通常 1000 是第一个普通用户）
  - 这里记录了用户实际修改过的配置值
- **查看某个配置的详细信息**（名称、描述、权限等）：
  ```bash
  dde-dconfig get -a <appid> -r <resource> -k <key> -m name
  dde-dconfig get -a <appid> -r <resource> -k <key> -m discription
  ```

### 已知的 DConfig appid

常见组件的 appid（`-a` 参数）：
- `org.deepin.dde.control-center` — 控制中心
- `org.deepin.dde.daemon` — dde-daemon（含 XSettings 等子 resource）
- `org.deepin.dde.appearance` — 外观设置
- `org.deepin.dde.clipboard` — 剪贴板
- `org.deepin.dde.dock` — dock 栏
- `org.deepin.dde.file-manager` — 文件管理器
- `org.deepin.ds.dock` — dde-shell dock 插件
- `dde-launchpad` — 启动器

### 排查配置问题

当用户遇到某个设置不生效的问题时：
1. 用 `dde-dconfig get` 检查当前配置值是否正确
2. 检查 `/var/lib/dde-dconfig-daemon/.config/<uid>/` 下的缓存文件是否有异常
3. 用 `dde-dconfig reset` 重置为默认值

---

## deepin-service-manager 日志

deepin-service-manager 管理 D-Bus 插件式服务，它托管的服务包括：
- **dde-appearance** 安装的个性化服务（外观、主题等）
- **dde-services** 安装的 XSettings 等服务

查看 deepin-service-manager 整体日志（需要 root）：
```bash
# 查看已有日志
sudo journalctl -b 0 /usr/bin/deepin-service-manager --no-pager
# 实时跟踪（会阻塞终端，用 Ctrl+C 退出）
sudo journalctl -f /usr/bin/deepin-service-manager
```

排查 D-Bus 服务加载/激活问题时，这个日志非常关键。

---

## dde-daemon 调试指南

dde-daemon 是 Go 编写的。它常驻的进程只有两个：
- `dde-session-daemon` — 用户级会话守护进程
- `dde-system-daemon` — 系统级守护进程

其他子功能大部分是以 D-Bus 服务的形式由 systemd 激活。如果要查看某个 D-Bus 服务的日志：
```bash
journalctl --user -u org.deepin.dde.Display1 -b 0 --no-pager
```

### 用 debug 模式重启 dde-daemon

```bash
# 停掉常驻进程，然后手动带 debug 启动查看日志
systemctl --user stop dde-session-daemon
DDE_DEBUG_LEVEL=debug /usr/lib/deepin-daemon/dde-session-daemon
```

### dde-daemon 安装的独立二进制工具

这些是 dde-daemon 提供的命令行工具，可以通过 `journalctl --user /usr/lib/deepin-daemon/<binary>` 查看其日志：

| 二进制路径 | 作用 |
|-----------|------|
| `/usr/lib/deepin-daemon/backlight_helper` | 背光亮度辅助 |
| `/usr/lib/deepin-daemon/dde-lockservice` | 锁屏服务 |
| `/usr/lib/deepin-daemon/dde-greeter-setter` | 登录界面设置 |
| `/usr/lib/deepin-daemon/langselector` | 语言选择器 |
| `/usr/lib/deepin-daemon/soundeffect` | 音效播放 |
| `/usr/lib/deepin-daemon/grub2` | GRUB 引导配置 |
| `/usr/lib/deepin-daemon/search` | 搜索服务 |
| `/usr/lib/deepin-daemon/default-file-manager` | 默认文件管理器 |
| `/usr/lib/deepin-daemon/default-terminal` | 默认终端 |

### dde-daemon 子服务及故障现象

| 服务名 | 功能 | 出问题时的表现 |
|--------|------|--------------|
| `org.deepin.dde.Display1` | 显示管理（分辨率、多屏） | 分辨率异常、多屏不识别 |
| `org.deepin.dde.Accounts1` | 账户管理 | 用户头像不显示、账户设置异常 |
| `org.deepin.dde.Bluetooth1` | 蓝牙 | 蓝牙无法配对、设备列表为空 |
| `org.deepin.dde.Power1` | 电源管理 | 电池百分比不准、合盖不休眠 |
| `org.deepin.dde.Gesture1` | 手势（触控板） | 三指/四指手势不工作 |
| `org.deepin.dde.Timedate1` | 时间日期 | 时间不准、时区设置无效 |
| `org.deepin.dde.LockService1` | 锁屏服务 | Super+L 不锁屏 |
| `org.deepin.dde.Network1` | 网络 | 网络列表为空 |
| `org.deepin.dde.SoundEffect1` | 音效 | 系统通知无声 |

---

## dde-session-ui 独立二进制工具

dde-session-ui 安装的工具（大部分不是常驻进程，按需启动）：

| 二进制路径 | 作用 |
|-----------|------|
| `/usr/bin/dde-hints-dialog` | 提示对话框 |
| `/usr/bin/dde-license-dialog` | 许可协议对话框 |
| `/usr/bin/dde-pixmix` | 图片混合工具 |
| `/usr/bin/dde-switchtogreeter` | 切换到登录界面 |
| `/usr/bin/dde-wm-chooser` | 窗口管理器选择器 |
| `/usr/bin/deepin-login-reminder` | 登录提醒 |
| `/usr/lib/deepin-daemon/dde-welcome` | 欢迎界面 |
| `/usr/lib/deepin-daemon/dde-lowpower` | 低电量提示 |
| `/usr/lib/deepin-daemon/dde-warning-dialog` | 警告对话框 |
| `/usr/lib/deepin-daemon/dde-blackwidget` | 黑屏组件 |
| `/usr/lib/deepin-daemon/dde-bluetooth-dialog` | 蓝牙对话框 |
| `/usr/lib/deepin-daemon/dde-suspend-dialog` | 休眠对话框 |
| `/usr/lib/deepin-daemon/dde-touchscreen-dialog` | 触摸屏对话框 |

查看这些工具的日志：
```bash
journalctl --user --no-pager -b 0 /usr/lib/deepin-daemon/dde-welcome
```

---

## 组件故障 → 问题现象对照

| 问题现象 | 首先排查 | 其次排查 |
|---------|---------|---------|
| 桌面黑屏/无元素 | dde-shell | dde-session、显卡驱动 |
| dock 栏消失/异常 | dde-shell（重启 `dde-shell@DDE.service`） | dde-tray-loader 日志 |
| 托盘图标不显示 | dde-tray-loader（看文件日志） | dde-network-core、dde-daemon |
| 控制中心闪退 | dde-control-center | D-Bus 服务、DConfig 配置 |
| 应用启动器打不开 | dde-launchpad | dde-application-manager |
| 锁屏不工作/黑屏 | dde-session-shell | dde-daemon LockService1 |
| 剪贴板不工作 | dde-clipboard | D-Bus 服务 |
| 主题切换失败 | dde-appearance | DConfig 配置 |
| 系统更新失败 | 模态更新日志或 lastore-daemon | 控制中心 GUI 日志 |
| D-Bus 报错 | deepin-service-manager | 相关 D-Bus 服务 |
| 手势不工作 | dde-daemon Gesture1 | 输入设备驱动 |
| 蓝牙问题 | dde-daemon Bluetooth1 | bluez 服务 |
| 电源/电池问题 | dde-daemon Power1 | upower 服务 |
| 通知不显示 | dde-shell（通知插件） | dde-session-ui |

---

## 排查流程建议

1. **确定涉及的组件** — 根据问题现象判断是哪个组件负责（参考上方对照表）
2. **尝试重启组件** — 桌面相关用 `systemctl --user restart dde-shell@DDE.service`，其他服务用对应的 `systemctl --user restart`
3. **查看对应服务的 journal 日志** — `journalctl --user -u <service> -b 0` 或 `journalctl --user /usr/bin/<binary>`
4. **检查 DConfig 配置** — 用 `dde-dconfig get` 检查相关配置值是否异常
5. **如果 journal 没有足够信息** — 检查是否有文件日志、是否需要开启 debug 日志
6. **D-Bus 相关问题** — 查看 `dbus-daemon` 日志和相关服务是否激活

### 如果无法解决

当排查后仍无法定位问题时，**告诉用户哪段日志可能是问题点**，让他把关键日志片段发给 DDE 团队分析。引导用户到具体的错误行，而不是给一大堆日志。

### 快速诊断命令

```bash
# DDE 会话整体状态
systemctl --user status dde-session-manager

# 所有 DDE 相关服务状态
systemctl --user list-units --type=service | grep -i "dde\|deepin"

# 最近的 DDE 错误
journalctl --user -b 0 -p err --no-pager | grep -i "dde\|deepin"

# D-Bus 服务注册情况
busctl --user list | grep -i "deepin\|dde"
```

---

## D-Bus 服务名索引

- `org.deepin.dde.control-center` / `org.deepin.dde.ControlCenter1` → dde-control-center
- `org.deepin.dde.Appearance1` / `com.deepin.wm` → dde-appearance
- `org.desktopspec.ApplicationManager1` → dde-application-manager
- `org.deepin.dde.Clipboard1` / `org.deepin.dde.daemon.Clipboard1` → dde-clipboard
- `org.deepin.dde.LockFront1` / `org.deepin.dde.ShutdownFront1` → dde-session-shell
- `org.deepin.dde.Display1` → dde-daemon (显示子模块)
- `org.deepin.dde.Bluetooth1` → dde-daemon (蓝牙子模块)
- `org.deepin.dde.Power1` → dde-daemon (电源子模块)
- `org.deepin.dde.XSettings1` → dde-services
- `org.desktopspec.ApplicationUpdateNotifier1` → dde-application-manager (更新通知)
- `org.deepin.service.manager` → deepin-service-manager
