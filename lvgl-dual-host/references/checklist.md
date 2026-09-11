# 落地顺序

参考实现：产品表「只读参考」列指向的已落地 `<demo>`（`ui_app.c` / `ui_port.h` / `ui_board.c`）及其 Guider `custom/`。只读，禁止改。

1. 读产品表或让用户点名：方案、Kconfig、`<demo>`、Guider 工程。
2. 以**当前产品**板上的 `ui_app.c` 与用户指定的 Guider **谁新听用户**，先对齐再抽 port。不要按参考产品的功能清单扩需求。
3. 在 `<demo>` 落地 `ui_port.h` / `ui_board.c` / `guider.mk`；逻辑仍只在这份 `ui_app.c`。
4. Guider：`custom.c` 只调 `ui_app_init()`；加上 `ui_port_sim.c`；删掉 `custom/ui_app.c`（若有）。
5. `custom.mk` include `guider.mk`；`AIC_UI_DIR` 指向该 `<demo>`。
6. 切图：规范源在 `<demo>/assets/`，相对路径对齐；动态路径缓冲 ≥ 64。
7. 该产品 defconfig **只开表里的那一个** LVGL demo Kconfig。
8. 编译用组合 skill；`*.d` 串工程则删 `lvgl-simulator/build` 再全量。

## 检查

- [ ] 没有改只读参考产品
- [ ] 没有把两产品合成一份 UI
- [ ] 没有整棵 Guider 进 SCons
- [ ] Guider 无第二份 `ui_app.c`
- [ ] 量产已链符号语义未改
- [ ] 未擅自 commit / 烧录
