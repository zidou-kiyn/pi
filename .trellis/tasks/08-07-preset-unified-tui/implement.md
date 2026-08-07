# Implementation record

Repo: /home/heixiaohu/桌面/pi-preset

## Changes

- `src/manifest.ts`: REQUIRED_PACKAGES 移除 chrome-devtools/playwright，新增 `npm:pi-context-view`、`npm:pi-btw`（仍为 13 个）；新增 `OPTIONAL_PACKAGES`（OptionalPackage: source/label/description）。
- `src/plan.ts`: `plan(options?: PlanOptions)` 支持 `extraPackages`，按 packageIdentity 去重后并入 desired 集合。
- `src/optional-packages-ui.ts`（新）: OptionalPackagesComponent checkbox 组件 + `selectOptionalPackagesWithUi`；已安装项预勾选且锁定（additive-only）。
- `src/preset-sync-run.ts`（新）: 原 extensions/preset-sync.ts 逻辑 + TUI 模式下同步前弹可选扩展 checklist；Esc 取消整个 sync。
- `extensions/preset-skills-sync.ts` → `src/skills-sync-run.ts`、`extensions/preset-models-add.ts` → `src/models-add-run.ts`（git mv，去掉各自 registerCommand，用户可见字符串改为 pi-preset 前缀）。
- `extensions/preset-sync.ts` 删除。
- `extensions/pi-preset.ts`（新）: 单一 `/pi-preset` 命令。TUI 用 DescribedSelectComponent 三项主菜单（sync/skills/models）；RPC 用 ctx.ui.select；print/json 输出 sync dry-run plan。
- 测试: import 路径更新；json-patch-plan.test.ts 新增可选包测试（默认不计划、勾选后计划、与必装包去重）。
- README 全面更新（Bootstrap、Optional extensions 章节、13+2 扩展表、命令名）。

## Verification

- `node --test test/*.test.ts`: 107 pass, 0 fail（基线 106，新增 1）。
- tsgo strict typecheck（src + extensions）: 通过。
- `scripts/scan-secrets.sh`: clean。
- `node -e import('./extensions/pi-preset.ts')`: default export 加载正常。
- 测试依赖 @earendil-works/* 通过 node_modules 软链到 ~/桌面/pi monorepo（已 build tui/coding-agent/ai/agent/protocol dist），node_modules 已 gitignore。
