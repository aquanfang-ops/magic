# 为什么不能把整棵 Guider 当板端 LVGL

| 差异 | 模拟器（Guider） | 板端（Luban-Lite） |
|------|------------------|-------------------|
| LVGL 核 | Guider 自带，给 SDL 用 | SDK / RT-Thread 那份 |
| 宿主 | SDL + 键盘 | `ui_init` → 串口按键 / FINSH / 板级外设弱符号 |
| 资源 | `A:` → 模拟器 assets（或指向 SDK assets） | SConscript 把 `<demo>/assets/` 打进 rodata |
| 工程位置 | 常在 Guider Projects 目录，**不在** SDK git | 必须在 SDK git |
| `generated/` | NXP 许可，设计脚手架 | **不能**当板端源码仓库 |

硬链接 / 整树拷进 SCons：换机器即废，且会把错误的 LVGL 核和许可文件编进板端。

能共用的只有：**页面逻辑**（SDK 里的 `ui_app.c` 等）和 **相对路径相同的切图文件名**。Guider 只提供 SDL 进程，用 Makefile `VPATH` 编译 SDK 路径上的 `.c`。
