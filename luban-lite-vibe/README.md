# luban-lite-vibe

面向**编码代理（agent）** 的 ArtInChip（匠芯创）Luban-Lite 规程：编译 → 串口进升级 → 烧录 → 串口验证。

不绑定某一家 IDE 或某一家模型。只要代理能加载 `SKILL.md`，按各工具自己的 skills 目录放入即可。

非官方、与匠芯创无隶属。不包含 SDK、AiBurn 或 `upgcmd.exe`。

## 安装

克隆后目录名保持 `luban-lite-vibe`，放到**当前代理**的个人或项目 skills 路径（以该工具文档为准），例如：

```text
<agent-skills>/luban-lite-vibe/
├── SKILL.md
└── references/
```

```bat
git clone https://github.com/aquanfang-ops/luban-lite-vibe.git
```

新开一轮对话后生效。用户说「编译」「烧录」「进升级」等即可触发；未点名方案时先列板并停等。

## 本机准备

- Luban-Lite SDK（含 `tools/env`）
- Windows：Python 3 + `pyserial`；环境变量 `AIC_UPGCMD` 指向 AiBurn 的 `upgcmd.exe`
- Linux：`source tools/onestep.sh`；SDK 自带 `tools/scripts/upgcmd`

不要把 COM、盘符、板名、账号写进 skill。

## 仓库结构

```text
SKILL.md              # 门禁与流程（代理先读这个）
references/
  windows.md          # Windows 环境与闭环
  linux.md            # Linux / WSL 差异
  flash.md            # USB / UART / 只烧 os
  serial.md           # 预检、aicupg、capture
  pitfalls.md         # 陷阱速查
```

## 许可

MIT。厂商工具与 SDK 仍按各自协议使用。
