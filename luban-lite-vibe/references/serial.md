# 串口：发现、预检、aicupg、capture、物理进升级

Windows 用主机 Python 3 + pyserial。临时脚本写入 `$env:TEMP`，用完即删。写文件用 `[IO.File]::WriteAllText(..., UTF8Encoding($false))`，不要 `Set-Content -Encoding UTF8`（PS 5.1 带 BOM）。

USB 烧录时调试口 ≠ 升级 USB。UART 烧录时调试口就是 uart0。

## 发现端口

必须无沙箱。先打印描述/`hwid`，只对 USB-UART 预检。

```powershell
python -c "from serial.tools import list_ports
for p in list_ports.comports():
    print(p.device + '\t' + (p.description or '') + '\t' + (p.hwid or ''))"
```

硬禁：对 `list_ports` 全量 `Serial.open`。描述或 `hwid` 含 `BLUETOOTH` / `BTHENUM` 的口永不 open（会挂死数十秒～数分钟）。

无 USB-UART 则失败退出，先解决驱动 / 接线 / 占用。

## 预检（发 aicupg 前必做）

通过：板端有回显（提示符或可打印字符）。失败不要发 aicupg。先 `$rc = $LASTEXITCODE` 再删临时文件。

```powershell
$PORT = $env:AIC_SERIAL_PORT
$py = @'
import sys, time, serial
port = sys.argv[1]
ser = serial.Serial(port=port, baudrate=115200, timeout=0.2)
try:
    ser.reset_input_buffer(); ser.write(b"\r\n")
    end = time.time() + 2; buf = ""
    while time.time() < end:
        data = ser.read(4096)
        if data:
            s = data.decode("utf-8", "replace"); buf += s
            sys.stdout.write(s); sys.stdout.flush()
finally:
    ser.close()
if not buf.strip():
    print("\n[SERIAL FAIL] 串口无回显"); sys.exit(1)
print("\n[SERIAL OK]")
'@
$tmp = Join-Path $env:TEMP ("aic_serial_precheck_{0}.py" -f $PID)
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText($tmp, $py, $utf8NoBom)
python $tmp $PORT
$code = $LASTEXITCODE
if ($null -eq $code) { $code = 1 }
Remove-Item $tmp -Force -ErrorAction SilentlyContinue
if ($code -ne 0) { exit 2 }
```

`Invoke-AicPy` 写法见 [windows.md](windows.md)：stdout 先收集再打印，函数只返回 int。

## 发送 aicupg

```powershell
$py = @'
import sys, time, serial
port, cmd, timeout = sys.argv[1], sys.argv[2], float(sys.argv[3])
ser = serial.Serial(port=port, baudrate=115200, timeout=0.2)
try:
    ser.reset_input_buffer(); ser.write((cmd + "\r\n").encode())
    end = time.time() + timeout; buf = ""
    while time.time() < end:
        data = ser.read(4096)
        if data:
            s = data.decode("utf-8", "replace"); buf += s
            sys.stdout.write(s); sys.stdout.flush()
finally:
    ser.close()
low = buf.lower()
if ("upgrade" in low) or ("upg" in low) or ("reboot" in low):
    print("\n[AICUPG OK]"); sys.exit(0)
print("\n[AICUPG ?] 未检测到升级回显"); sys.exit(2)
'@
```

调用：`python $tmp $PORT "aicupg" 5`。退出码：`0` 见到 upgrade/reboot；`2` 不确定，最多再试 1 次，仍 `2` 则转物理方式。

发完必须 `ser.close()`，再把 COM 交给 `upgcmd`。

## capture 与 reset

until 默认 `Welcome`（或 `Board:` / `Run APP`）。禁用 `Luban-Lite` / `aic`。避免 `>`。

USB：先 `Start-Process` capture，睡 300ms，再 `shcmd reset`。  
UART：先 `upgcmd -u $PORT -b $BAUD shcmd reset`，再开 capture。

```powershell
$py = @'
import sys, time, serial
port, log, sec, until = sys.argv[1], sys.argv[2], float(sys.argv[3]), sys.argv[4]
ser = serial.Serial(port=port, baudrate=115200, timeout=0.2)
buf = ""; rc = 1; end = time.time() + sec
try:
    with open(log, "w", encoding="utf-8", errors="replace") as f:
        while time.time() < end:
            chunk = ser.read(4096)
            if not chunk: continue
            s = chunk.decode("utf-8", "replace"); buf += s
            f.write(s); f.flush(); sys.stdout.write(s); sys.stdout.flush()
            if until and until in buf:
                rc = 0; break
finally:
    ser.close()
sys.exit(rc)
'@
```

日志写 `.log/boot_<时间戳>.log`。成功以 log 含 until 为准：

```powershell
$hit = $false
if (Test-Path $LOG) {
  $bootTxt = Get-Content $LOG -Raw -ErrorAction SilentlyContinue
  if ($bootTxt -and ($bootTxt -like ("*{0}*" -f $UNTIL))) { $hit = $true }
}
```

`$capProc.ExitCode` 仅为辅证。UART 漏 boot 报 WARN，不要判烧录失败。

## 物理方式进升级（兜底）

| 方式 | 操作 |
|------|------|
| 按键 | 按住 UPG/Boot，再 Reset 或上电，1–2s 松手 |
| 跳线 | 短接 UPG，上电或复位，1–2s 断开 |
| 电源 | 拔电再插（部分板缺省进 UPG） |
| AiBurn GUI | 主机端进升级后关掉 GUI 再命令行烧 |

之后 USB `-l` 或 UART `-u $PORT -l` 应见 `Boot ROM`。

## CH340 假死

默认先当主机 USB-UART 卡死，不是板死机。

| 现象 | 处置 |
|------|------|
| 打不开 COM / 读写立即失败 | 关占用程序 → 拔插**调试串口 USB**（不是升级 USB）→ 重新发现 COM（编号可能变）→ 再预检 |
| 能 open，发 `\r\n` 完全无回显 | 更像板端挂死：板 Reset，查 crash |

刚烧完就预检失败：立刻拔插调试串口，不要轮询数十次。一次脚本拿住串口做到底。未确认联网前慎用板端 `ping`。
