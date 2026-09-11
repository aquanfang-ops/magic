# magic

面向编码代理（agent）的 skill 仓库。不绑定某一家 IDE 或某一家模型。只要代理能加载 `SKILL.md`，按各工具的 skills 目录安装即可。

每个子目录是一个独立 skill：

```text
<skill-name>/
├── SKILL.md
└── references/    # 可选
```

用 CC Switch 时，仓库填：

```text
aquanfang-ops/magic
```

分支 `main`。应识别到多个技能（每个含 `SKILL.md` 的子目录算一个）。

## 已收录

| 目录 | 说明 |
|------|------|
| [luban-lite-vibe](luban-lite-vibe/) | 匠芯创 Luban-Lite：编译 → 串口进升级 → 烧录 → 串口验证 |
| [gui-guider-simulator](gui-guider-simulator/) | NXP GUI Guider LVGL 模拟器：32 位 MinGW 编译 → 运行 |

## 许可

各 skill 目录内许可为准；未单独声明的按本仓库 MIT。
