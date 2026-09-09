# 陷阱速查

| 陷阱 | 现象 | 处置 |
|------|------|------|
| 未注入 env | `scons.bat` / 工具链找不到 | 每个新 Shell 重注 |
| 系统/Git Bash `scons` | ModuleNotFound / 错 Python | 用 SDK 内 `scons.bat` |
| SDK `upgcmd.exe` 缺 DLL | `0xC0000135` | `AIC_UPGCMD` 指 AiBurn |
| 日常不在升级 | `-l` 无设备 | 先 `aicupg`；无 USB 再 `-u` |
| 无 USB 刷机当 USB 烧 | `-l` 一直空 | `AIC_UPG_TRANSPORT=uart` |
| UART 未带 `-u` | 已 aicupg 仍无设备 | `upgcmd -u $PORT -b 115200 -l` |
| UART 整包 | 烧录数十分钟 | 日常 `image … os` |
| 串口预检无回显 | 发 `\r\n` 无输出 | 接线、波特率、COM、占用；关沙箱 |
| aicupg 未进升级 | 预检过、无 Boot ROM | 重试 1 次；再物理进升级 |
| 串口被占用 | Access denied / PermissionError | 关串口终端 |
| CH340 假死 | 突然无法 open | 拔插调试串口 USB；重新发现 COM |
| USB 复位后才 capture | 漏 boot | USB：先 capture 再 reset |
| UART 先 capture 再 reset | Failed to open COM | UART：先 reset 再 capture |
| 第一段 ERROR | Pipe/claim 失败 | 看是否已到 U-Boot/Bootloader |
| 阶段名 | 等 Bootloader 却是 U-Boot | 两者都认 |
| until 用 `Luban-Lite`/`aic` | CRC 失败掉 tinySPL 仍判成功 | 用 `Welcome` / `Board:` / `Run APP` |
| until 含 `>` | capture 莫名失败 | 去掉 `>` |
| 根目录一堆 `log_*.txt` | upgcmd DEBUG | 闭环末尾删 |
| 多次独立 Shell | 极慢 | 能合并就同一进程；否则每步重注 env |
| AiBurn GUI 占用 | 命令行失败 | 关掉 GUI |
| 蓝牙 COM | 预检卡住数分钟 | 永不 open `BTHENUM` |
| 临时 py 带 BOM | 预检秒失败 | `UTF8Encoding($false)` |
| `$LASTEXITCODE` 被覆盖 | python 成功但 `$rc` 非 0 | 先取码再删文件 |
| `$null -ne 0` | log 已有 until 仍 VERIFY_FAIL | 以 log 为准 |
| 递归搜 upgcmd | Shell 长时间无输出 | 问用户路径 |
| 未列板就编译 | 用户只说「编译烧录」 | 先 `--list-noboot` 硬停 |
| 列板节选 / 纯文件名 | 用户以为没列 | 必须 `--list-noboot` 全量编号 |
| Invoke-AicPy 返回值污染 | 预检 OK 却判失败 | stdout 先收集，函数只返回 int |
| Select-String 无括号 | boot 成功仍失败 | 用 `-like` |
| fwc 用全名 | 0.1s 假成功 | 短名 `os`；校验日志与耗时 |
| `upgcmd -v -l` | 诊断挂死 | 杀进程；拔插升级 USB |
| 无事 clean | 增量变全量 | 只 `scons.bat -jN` |
