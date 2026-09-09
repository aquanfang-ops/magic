# Linux / WSL 主机差异

Windows 默认走 [windows.md](windows.md)。本页仅 Linux 差异。烧录策略与 until 规则仍遵守 SKILL.md 与 [flash.md](flash.md)。

## 环境

```bash
cd <SDK>
set +e +u +o pipefail
source tools/onestep.sh
```

`tools/onestep.sh` 不能在已开启的 `set -euo pipefail` 下直接 source（会静默失败）。调用 `_make_boot_and_app` 时保持 `set +u`，或 `_make_boot_and_app ""`。不要敲未展开的 alias `m` / `mb`。

烧录工具：`$SDK_PRJ_TOP_DIR/tools/scripts/upgcmd`（Linux 一般可用）。串口用系统 `python3` + termios，不必装 pyserial。sudo 凭据用环境变量 / `sudo -n`，不要写进 skill。

## 列板

优先与 `list` 一致。若环境提供 `scons --list-noboot` 则用它并贴全量编号。否则：

```bash
ls target/configs/*_rt-thread_helloworld_defconfig 2>/dev/null \
  | sed 's|.*/||; s|_defconfig$||' | sort
tr -d '\r\n' < .defconfig 2>/dev/null || true
```

用户未点名方案时同样硬停。不要混入 `baremetal_bootloader`。

## 编译与镜像

```bash
_make_boot_and_app ""
SOLUTION=$(tr -d '\r\n' < .defconfig)
IMG=$(ls -1t "output/${SOLUTION}/images/"*.img 2>/dev/null | head -1)
# 同时检查 output/images/*.img，按 mtime 取最新
```

成功：`Luban-Lite is built successfully`。

## 串口

```bash
export AIC_SERIAL_PORT="${AIC_SERIAL_PORT:-$(ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null | head -1)}"
```

多口时预检选有回显的那个。访问 `/dev/tty*` 须无沙箱。CH340 `Errno 5` / dmesg `-110`：先 `usbreset` / sysfs `authorized`，勿当板死机。

预检 / aicupg / capture 用 termios 内嵌 `python3`（与 Windows pyserial 片段同逻辑：发 `\r\n` 或 `aicupg`，读回显）。临时文件放 `/tmp`，用完即删。

Linux USB 烧录验证顺序：先 capture 再 `shcmd reset`。无 USB 刷机时同样 `-u $PORT`。

## 清理

```bash
rm -f "$SDK_PRJ_TOP_DIR"/log_*.txt
```
