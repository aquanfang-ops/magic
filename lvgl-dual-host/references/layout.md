# 目录与契约

路径用 `<SDK>`、`<demo>`、`<Guider>`。`<demo>` 来自产品表，不要写死系列名。

## SDK（唯一逻辑与规范切图）

```text
<SDK>/packages/artinchip/lvgl-ui/aic_demo/<demo>/
  ui_app.c           # 页面逻辑
  ui_port.h          # A() / UI_FS_PREFIX / log/beep/kled/boot_audio/after_init
  ui_board.c         # ui_init、按键/电位器投递、FINSH、弱符号板级 API
  assets/            # 规范切图（进 rodata；先压再给 SCons）
  guider.mk          # 给模拟器 include；无盘符
```

`ui_port.h` 用宏区分宿主，例如模拟器 `UI_FS_PREFIX` 为 `"A:"`，板端为 `LVGL_DIR`（或产品表约定的前缀）。`A("foo.png")` 只拼相对名。

板级外设用**弱符号**（蜂鸣、灯、开机音等），模拟器可以空实现或在 `ui_port_sim.c` 里做 SDL 提示音 / 键盘。

## Guider（只当宿主）

```text
<Guider>/
  custom/
    custom.c         # custom_init 只调 ui_app_init()，不要再传 lv_ui *
    ui_port_sim.c    # SDL 蜂鸣 + 键盘 → ui_app_post_*
    custom.mk        # 理想情况只 include $(AIC_UI_DIR)/guider.mk
  lvgl-simulator/
    assets/          # 若未把 A: 指到 SDK，则与 <demo>/assets 相对路径同名（同步而来）
  generated/         # 勿当板端源；勿为「对齐 UI」手改后当主源
  source/            # 设计稿，不是运行时资源
```

**不要**在 `custom/` 再放 `ui_app.c`。

`guider.mk` 要点（值由环境 / 产品表解析）：

```makefile
# 在 <demo>/guider.mk，不要写死盘符
GEN_CSRCS += custom.c ui_port_sim.c ui_app.c
VPATH += :$(PRJ_DIR)/custom:$(AIC_UI_DIR)
CFLAGS += "-I$(AIC_UI_DIR)" -DUSE_SIM_HOST
```

不要 `wildcard custom/*.c`（会把误放的第二份 `ui_app.c` 编进来）。

GUI Guider 点 Generate 若覆盖 `custom.mk`，只把 `include $(AIC_UI_DIR)/guider.mk` 加回去。

## 切图

- 改图改 `<demo>/assets/`（或同步脚本的输入），再同步 / 重映射到模拟器。
- 相对路径必须与 `A("...")` 一致。
- 不要把 Guider 的 `custom/`、`generated/`、`source/` 打进板端。
- 给客户看效果：`simulator.exe` + `SDL2.dll` + 整份运行用 `assets/`，不要整棵工程 zip。
