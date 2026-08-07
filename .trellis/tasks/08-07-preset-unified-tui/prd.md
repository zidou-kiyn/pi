# PRD: pi-preset unified /pi-preset TUI command with optional packages

Repo: /home/heixiaohu/桌面/pi-preset (external to this workspace)

## Requirements (user)

1. `@narumitw/pi-chrome-devtools` 和 `pi-playwright` 不再默认安装，改为同步前多选框勾选（默认不勾选；已安装的显示为已勾选）。
2. 合并 `/preset-sync`、`/preset-skills-sync`、`/preset-models-add` 为单一 `/pi-preset` 命令，TUI 主菜单可视化操作。
3. 预设新增默认扩展：`npm:pi-context-view`、`npm:pi-btw`。

## Design decisions

- **manifest.ts**: `REQUIRED_PACKAGES` 保持为默认安装集合，新增 `pi-context-view`、`pi-btw`；新增 `OPTIONAL_PACKAGES`（chrome-devtools、playwright），带每项说明文字供 checkbox 渲染。
- **plan.ts**: `plan()` 接受可选参数 `extraPackages: string[]`（勾选的可选包），合并进包检查。可选包未勾选且未安装 → 不出现在 plan；已安装 → 计入 "already present"。preset 只做增量添加，永不移除包。
- **统一命令 `/pi-preset`**（新 `extensions/pi-preset.ts`）：
  - TUI 主菜单（ctx.ui.select 或 custom 组件）：
    1. Sync preset（先弹可选扩展 checkbox → 计算 plan → diff 确认 → apply）
    2. Install/refresh grilling skills（复用 runPresetSkillsSync 逻辑）
    3. Add model provider（复用 runPresetModelsAdd 逻辑）
  - 旧命令 `/preset-sync`、`/preset-skills-sync`、`/preset-models-add` 删除，仅保留 `/pi-preset` 和 `/vibrant-footer`。
  - 非 TUI 模式（print/json/rpc）：sync 输出 dry-run plan（可选包不含在内）；skills/models 子流程维持原有各自模式限制说明。
- **checkbox 组件**: 复用 models-wizard-ui.ts 中现有 ctx.ui.custom checkbox 模式。

## Acceptance criteria

- [ ] REQUIRED_PACKAGES 不含 chrome-devtools/playwright，含 pi-context-view/pi-btw。
- [ ] /pi-preset 在 TUI 下呈现三项主菜单；Esc 取消不写任何文件。
- [ ] Sync 前 checkbox 勾选 chrome-devtools/playwright 后进入 plan；已安装项预勾选；未勾选不移除已装包。
- [ ] 旧三个命令不再注册。
- [ ] README 更新。
- [ ] 现有测试更新并通过。
