---
name: gui-guider-simulator
description: >-
  Builds and runs NXP GUI Guider LVGL SDL simulator on Windows with the
  bundled 32-bit MinGW. Use whenever the user mentions GUI Guider,
  Guider, LVGL 模拟器, lvgl-simulator, mingw32-make, simulator.exe,
  or asks to compile/run a Guider-exported project—even if they do not
  attach the compile markdown. Hard-stop to list projects unless the
  user already named one. Requires $env:GG (GUI Guider install root).
  For any coding agent that can load SKILL.md.
---

# GUI Guider 模拟器编译

面向编码代理的可移植规程，不绑定某一家产品。Windows PowerShell 上：注入 32 位 MinGW → 停占用 → `mingw32-make` → 运行模拟器。

不要把盘符、GUI Guider 版本、工程名、账号写进可复用默认值。冲突时以用户当前指令为准。

## 何时读哪份参考

| 情况 | 先读 |
|------|------|
| 注入 PATH、标准编译、运行 | [references/windows.md](references/windows.md) |
| 报错对照 | [references/pitfalls.md](references/pitfalls.md) |

## 占位符（运行时解析）

- `<WORKSPACE>`：含多个 Guider 工程的目录（常见为 `GUI-Guider-Projects`，或本工作区里能扫到 `lvgl-simulator/Makefile` 的父目录）。
- `<PROJECT>`：用户**本对话点名**的工程文件夹名；或 `$env:GG_PROJECT`。
- `<PRJ>`：`<WORKSPACE>\<PROJECT>`
- `<SIM>`：`<PRJ>\lvgl-simulator`
- `<GG>`：`$env:GG`，其下须有 `environment\mingw\bin\mingw32-make.exe`。未设则向用户要路径，**禁止全盘搜索**。

合法工程至少有：`custom/`、`generated/`、`lvgl/`、`lvgl-simulator/Makefile`。日常改 `custom/`；不要为对齐 UI 去改 `generated/`。运行时资源在 `<SIM>\assets`（LVGL 盘符 `A:`），不是 `source/`。

## 硬停：未点名工程则本回合只列目录

「编译 / 跑模拟器 / 按文档编」**不等于**已选工程。

用户消息里没有工程名时：

1. 在 `<WORKSPACE>` 列出含子目录 `lvgl-simulator\Makefile` 的文件夹名（全量，禁止节选）。
2. **停止本回合**。不注入也能列；未确认前不编译、不启动 exe。

用户点名后记 `PROJECT`，再进入编译。

## 分步（每个新 Shell 都要重注 PATH）

```text
0. 列工程硬停（未点名则单独一回合）
1. 注入 <GG>\environment\mingw\bin 到 PATH 最前；gcc -dumpmachine 必须含 i686
2. Stop-Process simulator（没有进程也没关系）
3. cd <SIM>；只跑一个 mingw32-make（日常增量，勿无事 clean）
4. 成功：日志有 Linking simulator.exe 且 exit 0
5. 运行必须在 <SIM>：.\simulator.exe（保留 assets + SDL2.dll）
```

- 不要用系统 / MSYS / 64 位 gcc，不要用 Git Bash 编这个模拟器。
- 不要在 `<PRJ>` 根目录 `make`。
- 同一 `<SIM>` 禁止并行两个 `mingw32-make`。
- `cp_lib` / `cp` 不是命令且 `Error 1 (ignored)`：**不当失败**。
- 改 `lv_conf.h` / 清过 `build/` 才允许 `mingw32-make clean`。
- 全量重编可能数分钟：等 exit，不要按固定 30s 当失败。
- 访问进程 / 写构建产物须无沙箱。
- 临时脚本用完即删，不要留在 `<PRJ>` / `<SIM>`。

## 完成声明

必须写实：编了哪个 `<PROJECT>`、是否 `Linking simulator.exe` 且 exit 0、是否已启动、画面/资源是否可见、未验证项。禁止「应该好了」。
