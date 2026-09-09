# Windows 主机：环境与闭环

每次新 PowerShell 先注入 env。人用可开 `win_cmd.bat`，代理不要依赖 doskey。

## 环境注入

```powershell
$SDK = (Resolve-Path ".").Path
$env:SDK_PRJ_TOP_DIR = $SDK
$env:ENV_ROOT  = Join-Path $SDK "tools\env"
$env:PKGS_ROOT = Join-Path $env:ENV_ROOT "packages"
$env:RTT_ROOT  = Join-Path $SDK "kernel\rt-thread"
$env:PYTHONIOENCODING = "UTF-8"
$env:PATH = "$(Join-Path $SDK 'tools\env\tools\bin');$(Join-Path $SDK 'tools\env\tools\Python27\Scripts');$(Join-Path $SDK 'tools\env\tools\Python38');$env:PATH"
$SCONS = Join-Path $SDK "tools\env\tools\Python27\Scripts\scons.bat"

if (-not $env:AIC_UPGCMD) {
  Write-Host "请先设置 AIC_UPGCMD=指向主机 AiBurn 的 upgcmd.exe 完整路径"
  exit 2
}
$UPG = $env:AIC_UPGCMD
if (-not (Test-Path $UPG)) { Write-Host "AIC_UPGCMD not found: $UPG"; exit 2 }
$BAUD = if ($env:AIC_UPG_BAUD) { [int]$env:AIC_UPG_BAUD } else { 115200 }
```

- 不要用 SDK 内 `tools/scripts/upgcmd.exe`（常缺 Qt DLL，`0xC0000135`）。
- Windows 上 `scons --aicupg` 通常不可靠；走串口 `aicupg` + `AIC_UPGCMD`。
- `python -c "import serial; print(serial.__version__)"` 自检 pyserial。

## 列板（= doskey list）

```powershell
& $SCONS --list-noboot -C $SDK
if (Test-Path ".defconfig") {
  Write-Host ("当前 .defconfig: {0}" -f (Get-Content ".defconfig" -Raw).Trim())
} else {
  Write-Host "当前 .defconfig: (无)"
}
```

预期含 `Built-in configs:` 与编号。必须把完整列表贴给用户。

用户确认后：

```powershell
& $SCONS "--apply-noboot=${SOLUTION}_defconfig" -C $SDK
# 或：& $SCONS "--apply-noboot=<编号>" -C $SDK
Set-Content -Path ".defconfig" -Value $SOLUTION -NoNewline
```

备用（仅 `--list-noboot` 不可用）：`Get-ChildItem "target/configs/*_rt-thread_helloworld_defconfig"` 并自行编号，仍须全量。

## 编译

```powershell
& $SCONS -C $SDK -j 8
```

成功：`Luban-Lite is built successfully`。勿把 `scons --info` 放进闭环。勿无事 `scons -c`。改过 `.config` / `rtconfig.h` / Kconfig / 链接脚本 / 分区表后再 clean。

镜像：

```powershell
$SOLUTION = (Get-Content .defconfig -Raw).Trim()
$imgs = @()
if (Test-Path "output\images") { $imgs += Get-ChildItem "output\images\*.img" -EA SilentlyContinue }
if (Test-Path "output\$SOLUTION\images") { $imgs += Get-ChildItem "output\$SOLUTION\images\*.img" -EA SilentlyContinue }
$IMG = $imgs | Sort-Object LastWriteTime -Descending | Select-Object -First 1
```

## 标准闭环（同一进程可粘贴）

前置：`AIC_UPGCMD` 已设；主机 Python 3 + pyserial。用户未点名方案时只跑列板段并 exit。

临时 py 必须 UTF-8 无 BOM；python 输出先收进变量再 `Write-Host`，函数只 `return` 退出码。

```powershell
$ErrorActionPreference = "Continue"
$SDK = (Resolve-Path ".").Path
Set-Location $SDK
$env:SDK_PRJ_TOP_DIR = $SDK
$env:ENV_ROOT  = Join-Path $SDK "tools\env"
$env:PKGS_ROOT = Join-Path $env:ENV_ROOT "packages"
$env:RTT_ROOT  = Join-Path $SDK "kernel\rt-thread"
$env:PYTHONIOENCODING = "UTF-8"
$env:PATH = "$(Join-Path $SDK 'tools\env\tools\bin');$(Join-Path $SDK 'tools\env\tools\Python27\Scripts');$(Join-Path $SDK 'tools\env\tools\Python38');$env:PATH"
$SCONS = Join-Path $SDK "tools\env\tools\Python27\Scripts\scons.bat"
if (-not $env:AIC_UPGCMD) { Write-Host "set AIC_UPGCMD first"; exit 2 }
$UPG = $env:AIC_UPGCMD
if (-not (Test-Path $UPG)) { Write-Host "AIC_UPGCMD missing: $UPG"; exit 2 }
if (-not (Test-Path $SCONS)) { Write-Host "scons.bat missing"; exit 2 }
$BAUD = if ($env:AIC_UPG_BAUD) { [int]$env:AIC_UPG_BAUD } else { 115200 }
$TRANSPORT = if ($env:AIC_UPG_TRANSPORT) { $env:AIC_UPG_TRANSPORT.Trim().ToLower() } else { "auto" }

$script:Utf8NoBom = New-Object System.Text.UTF8Encoding $false
function Invoke-AicPy([string]$Code, [string[]]$PyArgs) {
  $tmp = Join-Path $env:TEMP ("aic_vibe_{0}_{1}.py" -f $PID, [guid]::NewGuid().ToString('N').Substring(0,8))
  [System.IO.File]::WriteAllText($tmp, $Code, $script:Utf8NoBom)
  $rc = 1
  try {
    $lines = & python $tmp @PyArgs 2>&1
    if ($null -ne $LASTEXITCODE) { $rc = $LASTEXITCODE }
    foreach ($line in $lines) { Write-Host $line }
  } finally {
    Remove-Item $tmp -Force -ErrorAction SilentlyContinue
  }
  return $rc
}

$SOLUTION = ""
if ($env:AIC_SOLUTION) { $SOLUTION = $env:AIC_SOLUTION.Trim() }
$curDef = ""
if (Test-Path ".defconfig") { $curDef = (Get-Content ".defconfig" -Raw).Trim() }
if (-not $SOLUTION) {
  Write-Host "=== scons --list-noboot ==="
  & $SCONS --list-noboot -C $SDK
  Write-Host ("当前 .defconfig: {0}" -f $(if ($curDef) { $curDef } else { "(无)" }))
  Write-Host "请用户按编号或方案名选择；确认前禁止编译/烧录"
  exit 1
}
Write-Host "已确认方案: $SOLUTION"
if ($curDef -and ($curDef -ne $SOLUTION)) {
  & $SCONS "--apply-noboot=${SOLUTION}_defconfig" -C $SDK
  if ($LASTEXITCODE -ne 0) { Write-Host "apply-noboot failed"; exit 1 }
  Set-Content -Path ".defconfig" -Value $SOLUTION -NoNewline
}

& $SCONS -C $SDK -j 8
if ($LASTEXITCODE -ne 0) { Write-Host "build failed"; exit 1 }

$imgs = @()
if (Test-Path "output\images") { $imgs += Get-ChildItem "output\images\*.img" -ErrorAction SilentlyContinue }
$solImgDir = Join-Path $SDK "output\$SOLUTION\images"
if (Test-Path $solImgDir) { $imgs += Get-ChildItem (Join-Path $solImgDir "*.img") -ErrorAction SilentlyContinue }
$IMG = $imgs | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if (-not $IMG) { Write-Host "no .img found"; exit 1 }

if (-not $env:AIC_SERIAL_PORT) {
  $env:AIC_SERIAL_PORT = (& python -c "from serial.tools import list_ports
def is_bt(p):
    d=((p.description or '')+' '+(p.manufacturer or '')+' '+(p.hwid or '')).upper()
    return ('BLUETOOTH' in d) or ('BTHENUM' in d)
cands=[]
for p in list_ports.comports():
    if is_bt(p): continue
    d=((p.description or '')+' '+(p.manufacturer or '')).upper()
    if any(k in d for k in ('CH340','CH341','CP210','FTDI','USB-SERIAL','USB SERIAL')):
        cands.append(p.device)
if not cands:
    cands=[p.device for p in list_ports.comports() if not is_bt(p)]
print(cands[0] if cands else '')").Trim()
}
if (-not $env:AIC_SERIAL_PORT) { Write-Host "无调试串口（已排除蓝牙 COM）"; exit 2 }
$PORT = $env:AIC_SERIAL_PORT

$script:UseUart = ($TRANSPORT -eq "uart")
function Get-UpgBase {
  if ($script:UseUart) { return @("-u", $PORT, "-b", "$BAUD") }
  return @()
}

# 预检 / aicupg / Wait-BootRom / 烧录 / capture：见 serial.md 与 flash.md
# 用数组拼接 upgcmd，勿包一层带 -p/-l 的函数（PS 会把 -p 当函数参数吃掉）
```

预检、aicupg、等 Boot ROM、烧录、验证的完整片段见 [serial.md](serial.md) 与 [flash.md](flash.md)。闭环末尾：

```powershell
Remove-Item (Join-Path $SDK "log_*.txt") -ErrorAction SilentlyContinue
```

## 代理注意

- Shell 访问 COM/USB 须关沙箱。
- 分步执行时每个调用重注 env。
- `until` 避免含 `>`。断言用 `-like "*$UNTIL*"`，不要 `Select-String -Pattern [regex]::Escape(...)`（无括号会绑错参数）。
- 找 `upgcmd` 只用环境变量或用户指定。
