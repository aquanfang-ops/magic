# Windows：环境与编译

每次新 PowerShell 先注入 PATH。人可用 GUI Guider IDE 的 Run，代理不要依赖 IDE 按钮。

## 环境注入

```powershell
$ErrorActionPreference = "Continue"
$WS = (Resolve-Path ".").Path
if (-not $env:GG) {
  Write-Host "请先设置 GG=GUI Guider 安装根（其下应有 environment\mingw\bin）"
  exit 2
}
$MingwBin = Join-Path $env:GG "environment\mingw\bin"
if (-not (Test-Path (Join-Path $MingwBin "mingw32-make.exe"))) {
  Write-Host "mingw32-make 不在: $MingwBin"
  exit 2
}
$env:Path = "$MingwBin;" + $env:Path
$env:PYTHONIOENCODING = "UTF-8"

$mach = (& gcc -dumpmachine 2>$null)
Write-Host "gcc -dumpmachine = $mach"
if ($mach -notmatch "i686") {
  Write-Host "当前 gcc 不是 i686，停止。请把 GUI Guider 的 mingw\bin 放到 PATH 最前"
  exit 2
}
```

## 列工程

```powershell
Get-ChildItem $WS -Directory | Where-Object {
  Test-Path (Join-Path $_.FullName "lvgl-simulator\Makefile")
} | ForEach-Object { $_.Name }
```

把完整名单贴给用户。未点名则 exit，禁止编译。

## 标准编译

前置：`$env:GG`；用户已点名 `$env:GG_PROJECT` 或本会话 `$PROJECT`。

```powershell
$ErrorActionPreference = "Continue"
$WS = (Resolve-Path ".").Path
if (-not $env:GG) { Write-Host "set GG first"; exit 2 }
$MingwBin = Join-Path $env:GG "environment\mingw\bin"
if (-not (Test-Path (Join-Path $MingwBin "mingw32-make.exe"))) { Write-Host "bad GG"; exit 2 }
$env:Path = "$MingwBin;" + $env:Path

$PROJECT = ""
if ($env:GG_PROJECT) { $PROJECT = $env:GG_PROJECT.Trim() }
if (-not $PROJECT) {
  Write-Host "=== 含 lvgl-simulator 的工程 ==="
  Get-ChildItem $WS -Directory | Where-Object {
    Test-Path (Join-Path $_.FullName "lvgl-simulator\Makefile")
  } | ForEach-Object { $_.Name }
  Write-Host "请用户点名工程名；确认前禁止编译"
  exit 1
}

$PRJ = Join-Path $WS $PROJECT
$SIM = Join-Path $PRJ "lvgl-simulator"
if (-not (Test-Path (Join-Path $SIM "Makefile"))) {
  Write-Host "不是 GUI Guider 模拟器工程: $PRJ"
  exit 2
}

Get-Process simulator -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
Set-Location $SIM
mingw32-make
$rc = $LASTEXITCODE
if ($rc -ne 0) { Write-Host "build failed: $rc"; exit $rc }
Write-Host "OK: $(Join-Path $SIM 'simulator.exe')"
```

成功：输出含 `Linking simulator.exe`（随后可能有 `Linking simulator.dll`），`$LASTEXITCODE -eq 0`。

日常增量：已注入 PATH 且已 `cd <SIM>` 后，先停 `simulator` 再 `mingw32-make`。默认可不加 `-j`。同一工程不要同时开两个 make。

必须全量时才 `mingw32-make clean`：改过 `<SIM>\lv_conf.h`、换过 MinGW、清过 `build/`、链接产物残缺。

## 运行

工作目录必须是 `<SIM>`，以便 `A:` 映射到 `.\assets`。

```powershell
Set-Location $SIM
.\simulator.exe
```

若 Makefile 提供：`mingw32-make run`。

交付最小集合：`simulator.exe` + `SDL2.dll` + 整个 `assets/`。不要只发 exe。不要打包 `build\`、`source\`、设计源 zip。

| 现象 | 处置 |
|------|------|
| 窗口黑屏 / 缺图 | 在 `<SIM>` 启动；查 `assets` 与代码 `A:` 路径 |
| 一闪退出 | dll 与 32 位 exe 同目录（`modules\SDL2` 或 `<SIM>` 根） |
| 按键无反应 | 先点窗口；键位以该工程自己的说明为准 |
