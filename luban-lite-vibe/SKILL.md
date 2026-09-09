---
name: luban-lite-vibe
description: >-
  Runs the ArtInChip (匠芯创) Luban-Lite compile → serial upgrade → flash →
  boot-verify loop. Use whenever the user mentions Luban-Lite, 匠芯创,
  ArtInChip, 编译, 烧录, 刷机, 进升级, aicupg, upgcmd, scons.bat, 串口验证,
  or asks to build/flash any luban-lite* SDK. Do not skip this skill for
  “编译烧录” shorthand. Hard-stop to list boards unless the user already
  named a solution. Windows is the default host; Linux uses the linux
  reference. For any coding agent that can load SKILL.md.
---

# Luban-Lite 编译烧录闭环

面向编码代理（agent）的可移植规程，不绑定某一家产品。本 skill 触发后按本文执行，用户不必再附带操作手册。

不要把板名、COM、盘符、镜像名、账号写进可复用默认值。冲突时以用户当前指令为准。

## 何时读哪份参考

| 情况 | 先读 |
|------|------|
| Windows 主机（默认） | [references/windows.md](references/windows.md) |
| Linux / WSL | [references/linux.md](references/linux.md) |
| 烧录细节（USB 两段 / UART / 只烧 os） | [references/flash.md](references/flash.md) |
| 串口预检、aicupg、capture、CH340 | [references/serial.md](references/serial.md) |
| 报错对照 | [references/pitfalls.md](references/pitfalls.md) |

执行某步前打开对应参考，不要凭记忆改命令。

## 占位符（运行时解析）

- `<SDK>`：用户给出的 Luban-Lite 根目录；或含 `tools/env` 的当前工作区。设 `$env:SDK_PRJ_TOP_DIR`。
- `<solution>`：用户**本对话点名**的方案；落盘后才读 `.defconfig`。
- `AIC_UPGCMD`：主机 AiBurn（或等价）下的 `upgcmd.exe`。未设则向用户要路径，**禁止**递归扫盘。
- `AIC_SERIAL_PORT`：动态发现的 USB-UART（`COMx` / `/dev/ttyUSB*`）。排除蓝牙。
- `AIC_UPG_TRANSPORT`：`usb` | `uart` | 未设=`auto`（先 USB `-l`，无 Boot ROM 再 `-u`）。
- `AIC_UPG_BAUD`：未设则 **115200**。普通 CH340 不要抬。
- 镜像：`output/images/*.img` 与 `output/<solution>/images/*.img` 按 mtime 取最新。
- `AIC_BOOT_UNTIL`：未设则 `Welcome`。禁用 `Luban-Lite` / `aic`（会误判 SPL/tinySPL）。

Windows 额外依赖：主机 Python 3 + `pyserial`。编译只用 SDK 内 `scons.bat`，不要用系统/`Git Bash` 的 `scons`。

## 硬停：未点名方案则本回合只列板

「编译 / 烧录 / 按文档闭环」**不等于**已选板。`.defconfig` 存在也不等于已确认。

用户消息里没有方案名或 `list` 编号时：

1. 注入 env（见对应 OS 参考）。
2. Windows：`& $SCONS --list-noboot -C $SDK`（必须与 doskey `list` 相同）。Linux：见 linux 参考。
3. 把**完整**编号列表 + 当前 `.defconfig` 贴给用户。禁止节选。
4. **停止本回合**。不编译、不烧录、不预检串口。

用户回复编号或方案名后，记 `SOLUTION`，必要时 `apply-noboot`，再进入编译/烧录。

## 分步（代理每次 Shell 都是新进程）

每个 PowerShell/bash 调用开头必须重注 env，并重新解析 `UPG` / `PORT` / `SOLUTION` / `IMG`。

```text
0. 列板硬停（用户未点名则单独一回合）
1. 编译：增量 scons -jN；勿无事 -c
2. 进升级 + 烧录：预检 → aicupg → 等 Boot ROM → USB 两段 / UART 一段
   日常只烧 os（短名）。空片/改分区/改 SPL：AIC_UPG_FULL=1
3. 验证 boot：USB 先 capture 再 reset；UART 先 reset 再 capture
4. 清理 SDK 根目录 log_*.txt
```

- 串口预检失败：先排线/占用/沙箱/CH340 假死，不要发 `aicupg`。
- `aicupg` 两次仍无 Boot ROM：停，让用户做物理进升级，不要空轮询。
- 访问 COM/USB 必须无沙箱。关 SecureCRT / XShell / AiBurn GUI。
- 临时 `.py`：UTF-8 **无 BOM**；先保存退出码再删文件。
- 永不 `Serial.open` 蓝牙口（描述/`hwid` 含 `BLUETOOTH` / `BTHENUM`）。
- 验证以 `.log/boot_*.log` 是否含 until 为准，不要只看 `$capProc.ExitCode`（`$null -ne 0` 在 PS 为真）。
- 只烧 os 时日志必须出现 `image.target.os`，且耗时明显大于 1s。过滤必须用短名 `os`，禁用全名 `image.target.os`。

## 高危先问用户

分区表、`.config` / defconfig、链接脚本、pinmux、时钟。

## 完成声明

必须写实，禁止「应该好了」：

- 改了什么（若有）
- 是否出现 `Burn ... successfully`
- partial 时日志是否含 `image.target.os`
- 串口验证引用 log 原文
- 是否已清理 `log_*.txt`
- 未验证项与剩余风险
