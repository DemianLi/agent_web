# agent_web：從 nexus-agent 抽出的 web 子集（暫時給同事摸索架構用）

這是 `nexus-agent` 的剪枝快照，只留 `web/` 與它往下依賴的 11 個 workspace 套件（`harness/packages/*` 的 `wire`、`core`、`plugin-commands`、`tools`、`sdk`、`subprocess`、`timeout`、`brand`、`plugin-workspace-changes`，以及 `harness/vendor/{cordis,loader}`）。完整的 harness app 不在這裡。內文提到的 issue 編號（`#1545` 之類）指向原倉庫，這裡沒有。

## 需求
Node `^22.19.0 || >=24`，pnpm 10.34.5（`packageManager` 已寫明）。

## 指令
```bash
pnpm install --frozen-lockfile
pnpm run typecheck
pnpm run lint
pnpm run build
pnpm run test                              # 全部套件（有 1 個已知失敗，見下）
pnpm --filter @nexus/web run test:guard    # 需要本機 Chrome
pnpm --filter @nexus/web run catalog       # 需要本機 Chrome，且要先有 git commit
```

## 已知限制
- `harness/packages/nexus-core` 的 `src/backend-copy.test.ts`「基座原生的 CompositeBackend 掛在路由上」在**未剪枝的原倉庫也同樣失敗**，不是這次剪枝造成的。
- `pnpm --filter @nexus/web run test:e2e` 不支援：它要啟動 `harness/apps/harness` 的 serve，這裡沒有那個 app。
- 為了讓測試跑得起來，額外放了幾個只讀的參照檔（測試會把它們當文字讀）：`docs/operations.md`、`docs/tool-catalog.md`、`harness/native/sandbox-spawner/proxy-vectors.json`、`harness/apps/harness/src/settings/turn-timeout.ts`、`harness/packages/nexus-plugin-plan-mode/src/index.ts`。後兩者不是完整套件，不屬於 workspace。
- 與原倉庫不同的地方：`package.json` 拿掉 harness app 與發布相關的 script，`typecheck` 改成只跑這批套件，`pnpm-workspace.yaml` 拿掉 `harness/apps/*`，`pnpm-lock.yaml` 重新產生，並修了 `web/src/shared/lib/tool-output.test.ts` 一處指到倉庫外的相對 import（原倉庫的試驗分支就是壞的）。
