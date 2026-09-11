# 切图：模拟器先做，再上板

两套目录、相对路径必须同名。**不要双向自动对拍。** 一次只认一个「本轮主场」，做完用单向晋升或拉回，避免后写的盖掉已签字的效果。

| 目录 | 角色 |
|------|------|
| `<Guider>/lvgl-simulator/assets/` | **日常迭代主场**（客户新效果先在这里跑通） |
| `<demo>/assets/` | **上板 / git / rodata 主场**（量产只认这份） |
| `<Guider>/source/` | 设计原图，不是运行时资源 |

相对路径与代码 `A("boot/foo.png")` 一致。动态路径用 `UI_FS_PREFIX`，缓冲 ≥ 64。

## 日常：先模拟器，再晋升上板

```text
客户切图（先压，不要把 source/ 原图当运行时）
  → 放进 lvgl-simulator/assets/<相对路径>
  → 改 SDK 里那份 ui_app.c（VPATH），模拟器里把效果做完
  → 客户点头后：单向晋升到 <demo>/assets/（同相对路径）
  → 板端 scons（进 rodata）
```

晋升示例（Windows，只覆盖较新文件）：

```bat
xcopy /E /Y /D /I "<Guider>\lvgl-simulator\assets\*" "<SDK>\packages\...\aic_demo\<demo>\assets\"
```

晋升前确认：该相对路径在模拟器里已经是签字效果。不要整棵 Guider zip 当板端资源。

给客户看模拟器：`simulator.exe` + `SDL2.dll` + 当时用的那份 `assets/`。

## 急单：直接上板，回头必须拉回

```text
压图 → 放进 <demo>/assets/ → 改 ui_app.c → 板端编/烧
  → 下一轮再开模拟器之前：单向拉回
     <demo>/assets/  →  lvgl-simulator/assets/
```

急单后若不去拉回，模拟器仍是旧图，下一次「先做效果」会把板端已改的图盖掉。

代理：用户说「十万火急 / 直接上板」走急单；说「先看模拟器 / 先做效果」走日常。不要两边同时当主场改。

## 禁止

- 自动双向 sync（必漂）
- 以 `source/` 或 `generated/` 为运行时图
- SCons 直接吃未压缩原图
- 硬链接两目录
- 急单改完板端资产却不拉回，又在模拟器里当主场改同一路径
