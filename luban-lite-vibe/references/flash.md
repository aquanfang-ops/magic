# 烧录：USB 两段 / UART 一段 / 只烧 os

前提：设备已在升级。日常先串口 `aicupg`（见 [serial.md](serial.md)）。

`upgcmd -l` **不带 `-u` 只枚举 USB 升级口**。部分系列无 USB 刷机，须 UART。

```powershell
& $env:AIC_UPGCMD -l
& $env:AIC_UPGCMD -u $env:AIC_SERIAL_PORT -b 115200 -l
```

慎用 `-v`（可挂死）。关 AiBurn GUI。

## USB 两段

```powershell
& $UPG -l                                    # 须 Boot ROM
& $UPG -p image $IMG.FullName                # 第一段：Pipe/disconnect 常正常
# 轮询至 U-Boot 或 Bootloader（Stage 1），两种阶段名都认
& $UPG -p image $IMG.FullName os             # 日常第二段；整包则去掉 os
& $UPG shcmd reset                           # Overflow / pipe error 常正常
```

| 步骤 | 现象 | 含义 |
|------|------|------|
| 第一段 | disconnect / Switching to new stage | updater 已注入 |
| 第一段 | Pipe error / claim_interface failed | 常仍正常；以随后 `-l` 是否 Stage1 为准 |
| `-l` | `U-Boot` 或 `Bootloader`，Stage 1 | 可第二段 |
| 第二段 | `Burn ... successfully!` | 成功 |
| reset | Overflow / pipe / No such device | 常正常 |

不要因第一段 stderr 有 `ERROR` 就放弃。

验证顺序：**先开 capture，再 `shcmd reset`**（升级口 ≠ 调试口）。

## 日常只烧 os

```text
upgcmd image <file> [fwc list]
```

查组件（勿写死名称）：

```powershell
& $env:AIC_UPGCMD -i $IMG.FullName
```

helloworld 类镜像常见 partition 短名 `os`。

| 场景 | 第二段（USB）/ 一段（UART） |
|------|---------------------------|
| 只改 OS/应用/驱动 | `image $IMG.FullName os`（默认） |
| 同时改 data | `os,data` |
| 改 UI 资源（rodata） | `os,rodata` |
| 空片 / 改分区 / 改 SPL | 不带过滤；或 `$env:AIC_UPG_FULL=1` |

硬约束：

- 过滤用短名 `os` → 日志出现 `Upgrade fwc: image.target.os`。USB 常十余秒；UART 115200 约两分钟、约 8–10 KB/s。
- 过滤用全名 `image.target.os` → 约 0.1s 假成功，OS 未更新。
- 必须同时确认 `image.target.os` 与耗时 > ~3s，不能只看 `Burn ... successfully!`。
- USB 不可跳过第一段（除非已在 Bootloader）。UART 可从 Boot ROM 直接 `image … os`（会先打 updater）。
- 勿对 Boot ROM 发无过滤整包（UART 会按 ~10KB/s 传完 rodata 等）。

## UART（无 USB 刷机或 auto 落到 uart）

一根 USB-UART 接 **uart0**。`$env:AIC_UPG_TRANSPORT=uart`。握手始终 115200 8N1。

`aicupg` 之后必须关掉 pyserial，再把同一 COM 交给 `upgcmd`。

```powershell
& $UPG -u $PORT -b $BAUD -l
& $UPG -u $PORT -b $BAUD -p image $IMG.FullName os
& $UPG -u $PORT -b $BAUD shcmd reset
# 再开 capture（同口不能与 upgcmd 重叠）
```

| 现象 | 含义 |
|------|------|
| `-l` 无 `-u` 无设备 | 改 `-u $PORT` |
| updater 后 Got CAN / no CSW / Switching to new stage | 切 Stage1，继续传 os |
| `Upgrade fwc: image.target.os` | 必须见到 |
| 先 capture 再 shcmd → Failed to open COM | 先 reset 再采 |

验证顺序：**先 `shcmd reset`，再 capture**。漏 until 时再预检提示符或目视屏，勿当烧录失败。

## auto 等 Boot ROM

```powershell
function Wait-BootRom([int]$Sec) {
  $deadline = (Get-Date).AddSeconds($Sec)
  $out = ""
  while ((Get-Date) -lt $deadline) {
    $b = Get-UpgBase
    $out = & $UPG @b -l 2>&1 | Out-String
    if ($out -match 'Boot ROM') { return $out }
    Start-Sleep -Milliseconds 200
  }
  return $out
}
```

- `uart` / `usb`：该模式等 10s。
- `auto`：先 USB 等 4s；无 Boot ROM 再改 `-u $PORT` 等 10s。

设备已在 Boot ROM：跳过预检与 aicupg，直接从本节 `-l` 开始。

## 日志污染

`upgcmd` 每次调用可能在 CWD 生成 `log_<时间戳>.txt`。闭环末尾：

```powershell
Remove-Item .\log_*.txt -ErrorAction SilentlyContinue
```
