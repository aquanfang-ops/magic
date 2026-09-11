# 陷阱速查

| 日志 / 现象 | 真因 | 处置 |
|-------------|------|------|
| `mingw32-make` 不是命令 | PATH 未注入 | 每个新 Shell 按 windows.md 前置 mingw\bin |
| `undefined reference` 且一堆 SDL | 64 位 gcc | `gcc -dumpmachine` 须 i686 |
| `Permission denied`（链接阶段） | `simulator.exe` 被占用 | `Stop-Process simulator` 再编 |
| `ar: unable to rename ... File exists` | 并行 make | 只留一个 make；必要时删残库再编 |
| `cp_lib` / `cp` 不是命令，`Error 1 (ignored)` | Windows 无 `cp`，Makefile 已忽略 | **不当失败** |
| 编了很久仍在 `Compiling .../lvgl/src/...` | 全量重编 | 等结束；改过 `lv_conf.h` 时属预期 |
| 新图代码里有、画面没有 | 只改了 `source/` | 资源以 `<SIM>\assets` 为准 |
| 一闪退出 | 缺 `SDL2.dll` 或位数不对 | dll 与 32 位 exe 同目录 |
| 黑屏 / 缺图 | 没在 `<SIM>` 启动 | `Set-Location $SIM` 再跑 exe |
| Git Bash 下编 | 官方不支持 | 用 PowerShell / cmd |
| 在工程根 `make` | Makefile 在 `lvgl-simulator/` | `cd <SIM>` |
| 未点名就开编 | 默认拿某一个工程 | 先列含 Makefile 的目录并停等 |
| 全盘搜 GUI Guider | 极慢 | 只用 `$env:GG` 或问用户 |
