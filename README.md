# agent_web

[nexus-agent](https://github.com/DemianLi/nexus-agent)（萬物皆可插件的 Deep Agents Harness，全 TypeScript）的 web 子集，
給想先摸索前端架構的人用。

## 內容

| 位置 | 職責 |
| --- | --- |
| `web` | Vite ＋ React 19 ＋ Tailwind v4 ＋ shadcn/ui |
| `harness/packages/nexus-wire` | harness 與 web 之間的傳輸型別 |
| `harness/packages/nexus-core` 等 | web 相依的 plugin 契約與工具套件 |
| `harness/vendor` | Cordis 上游原始碼 |

沒有完整的 harness app，所以不能起 `serve`，也不能和 agent 對話。這個倉庫只用來建置、測試、讀程式碼。
和原倉庫的差異見 [`AGENT_WEB.md`](AGENT_WEB.md)。

## 從哪裡開始讀

1. [`web/ARCHITECTURE.md`](web/ARCHITECTURE.md)：web 的整體結構。
2. [`web/COMPONENTS.md`](web/COMPONENTS.md)：元件清單與用法。
3. [`web/GLOSSARY.md`](web/GLOSSARY.md)：用詞對照。
4. `web/src`：原始碼；`web/guard`、`web/catalog`、`web/e2e`：瀏覽器測試與元件目錄。

## 嵌入對話框 `<nexus-chat>`

要把對話框嵌進自己的系統（例如 3D 可視化），見 [`docs/embed-chat.md`](docs/embed-chat.md)：從哪裡開始讀、嵌入方式、用宿主頁工具讓對話控制宿主頁。

## 如何使用

需要 Node `^22.19.0 || >=24`、pnpm 10.34.5。

```bash
pnpm install --frozen-lockfile
pnpm run typecheck
pnpm run lint
pnpm run build
pnpm run test
```

- `pnpm run test` 會有 1 個已知失敗（`nexus-core` 的 `backend-copy.test.ts`），原倉庫也一樣。
- 瀏覽器測試需要本機 Chrome：`pnpm --filter @nexus/web run test:guard`；元件目錄 `... run catalog` 另外要先有 git commit。
- `test:e2e` 不支援，它需要完整的 harness app。
- 開發 web：`pnpm dev`（`vite build --watch`，輸出在 `web/dist`）。
- 新增套件或 subpath 後跑 `pnpm run gen-tsconfig-paths` 更新別名。
