---
name: lvgl-dual-host
description: >-
  Sets up and maintains one LVGL UI source compiled for two hosts: NXP
  GUI Guider SDL simulator and Luban-Lite board (SCons / rodata assets).
  Use when the user mentions 一份源两边编, 双宿主, ui_app.c, ui_port,
  custom.mk, VPATH, Guider 编 SDK 源码, 模拟器与板端同一套 UI, 切图两套目录,
  先模拟器后上板, 切图晋升, 急单上板,
  or asks to split/port UI so both the simulator and the board run the
  same logic. Do not use for compile-or-flash-only (luban-lite-vibe),
  simulator-build-only (gui-guider-simulator), or generic LVGL widget
  questions with no dual-host layout. Never invent product paths; read
  the SDK product table or hard-stop. For any coding agent that can
  load SKILL.md.
---

# LVGL 双宿主（一份逻辑，两边各编）

面向编码代理。一块板对应一份 UI。Guider 只当 SDL 宿主；板端只编 SDK git 里的那份源。

不要把产品名、盘符、Guider 工程名、SDK 绝对路径写进可复用默认值。产品对照表在 **SDK 仓库**，不在本 skill。冲突时以用户当前指令为准。

## 何时用 / 何时不用

先看用户是不是在改「源码住哪、两边怎么共用」，再决定。

**要用（读完本文再动手）：**

- 新系列要做成「模拟器 + 板端各编同一份 `ui_app.c`」
- 抽 `ui_port`、改 `custom.mk` / `VPATH`、消灭 Guider 里第二份 `ui_app.c`
- 切图相对路径、先模拟器后上板 / 急单先板再拉回、`UI_FS_PREFIX`、path 缓冲、Generate 冲掉 mk
- 用户说「照已落地的那套结构做」，但没让你只编/只烧

**不要用（改走别的 skill，或直接改业务代码）：**

| 用户实际要的 | 用哪个 |
|--------------|--------|
| 只编译 / 烧录 / 进升级 | `luban-lite-vibe` |
| 只编、只跑 Guider 模拟器 | `gui-guider-simulator` |
| 只改某一页按钮文案、颜色，两边工程已经按本规程接好 | 直接改 SDK 里那份 `ui_app.c`；切图按 assets-flow（日常先模拟器） |
| 问 LVGL 控件 API、无关双宿主 | 当普通 LVGL 问题 |

两边都要动时可以**组合**：本 skill 定布局 → 编模拟器走 `gui-guider-simulator` → 板端编/烧走 `luban-lite-vibe`。不要在本 skill 里再抄一遍 MinGW / scons / 烧录步骤。

## 何时读哪份参考

| 情况 | 先读 |
|------|------|
| 为什么不能整棵 Guider 进板端 | [references/architecture.md](references/architecture.md) |
| 文件怎么拆、`guider.mk` | [references/layout.md](references/layout.md) |
| 新切图：先模拟器或急单上板 | [references/assets-flow.md](references/assets-flow.md) |
| 落地顺序与检查清单 | [references/checklist.md](references/checklist.md) |

## 硬停

1. **没有产品表且用户没点名「SDK UI 目录 + 方案 + Guider 工程」** → 列出你在 SDK 里找到的 `UI_PRODUCTS.md` / `aic_demo/*_demo`，**本回合停止**，请用户补表或点名。禁止猜。
2. **Guider 工程未点名**（同系列可能有旧工程）→ 只列候选，停止。
3. 表里标了「只读参考」的产品：**禁止改**其 demo / defconfig / Kconfig / Guider。
4. 禁止把两个产品收成一份 `ui_app.c`。
5. 禁止把整棵 Guider 工程（`generated/`、`custom/`、`source/`）链进板端 SCons。
6. 不要主动 commit / 烧录，除非用户明确说。

产品表约定（SDK 内，有则读，无则请用户建）：`packages/artinchip/lvgl-ui/aic_demo/UI_PRODUCTS.md` 或同级等价文件。列：`id`、方案、Kconfig、SDK UI 目录、Guider 工程（须确认）、只读参考。

## 不可违反的结构

- **唯一逻辑源**：SDK git 里 `<demo>/ui_app.c`（及该 demo 下的 port）。Guider **不要**再留 `custom/ui_app.c`。
- **`ui_app_init(void)`** 用 `lv_screen_active()`，不要依赖 `gui_guider.h`。
- **路径**：`snprintf` 用 `UI_FS_PREFIX`，缓冲 ≥ 64。禁止写死 `A:` 或短 `path[24]`。
- **切图**：日常主场是模拟器 `assets/`，签字后**单向晋升**到 `<demo>/assets/` 再上板；急单先改 `<demo>/assets/`，再开模拟器前**单向拉回**。禁止双向自动对拍。`source/` 不当运行时资源。相对路径与代码一致。详见 [assets-flow.md](references/assets-flow.md)。
- **Guider Makefile**：`include` SDK 里的 `guider.mk`（或等价碎片），用 `VPATH` + 明确 `GEN_CSRCS`，不要 `wildcard custom/*.c`。`AIC_UI_DIR` / `<SDK>` 运行时解析，不要写死盘符。Generate 冲掉 `custom.mk` 后只恢复 include 那一行。
- **量产符号**：以该 demo **已经对外链接**的 API 为准，只抽 port、不改语义。参考产品的功能（多出来的模式/食谱）禁止硬抄。

## 编译怎么接

- 板端：用户确认方案后走 `luban-lite-vibe`（Windows 用 SDK 内 `scons.bat`）。
- 模拟器：走 `gui-guider-simulator`（`$env:GG`、i686、`cd lvgl-simulator`、`mingw32-make`）。
- `build/*.d` 里残留**别的工程**路径：删掉 `lvgl-simulator/build` 再全量，不要指望 Windows 上 `make clean` 的 `rm -rf`。

## 完成声明

写实：改了哪个 demo / 哪个 Guider 工程、逻辑源是否只剩 SDK 一份、`custom.mk` 是否 include 碎片、assets 是否单源、有没有动只读参考、未验证项。禁止「应该好了」。
