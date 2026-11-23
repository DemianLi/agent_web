# 嵌入對話框 `<nexus-chat>`

宿主頁（例如 3D 可視化系統）放一個元素就有對話框，不用 iframe；元件用 Shadow DOM，樣式兩邊互不影響。
`pnpm run build` 之後產物在 `web/dist/embed.js` 與 `web/dist/nexus-chat.d.ts`。要真的對話，得由原倉庫的 harness `serve` 供應這兩個檔並接後端。

## 從哪裡開始讀

1. [`web/src/app/embed/nexus-chat.d.ts`](../web/src/app/embed/nexus-chat.d.ts)：元素對宿主頁公開的全部介面（屬性、`hostTools`、`signIn`、事件）。
2. [`web/embed-demo/App.vue`](../web/embed-demo/App.vue)：Vue 3D 測試頁，示範 `context` 跟著選取、`hostTools` 宣告與 `nexus-host-tool` 處理。
3. [`web/src/app/embed/`](../web/src/app/embed/)：實作。進入點 `nexus-chat-element.tsx`，宿主頁工具 `host-tools.ts`，換票登入 `sign-in.ts`、`ticket-needed.ts`。
4. [`docs/operations.md`](operations.md)：serve 端的設定，看〈放在反向代理後面〉、〈嵌入方換票登入〉與設定表的 `host-tools` 列。
5. 本頁是摘要，完整版（開發時接遠端 serve、可覆寫的外觀、續期細節）在 [nexus-agent 的 `docs/embed-chat.md`](https://github.com/DemianLi/nexus-agent/blob/develop/docs/embed-chat.md)。

## 嵌入方式

宿主頁要跟 serve 同源：反向代理把 `/nexus/` 底下的請求去掉前綴轉給 serve，其餘轉給宿主頁。元件照 `embed.js` 自己的網址找 serve。

```html
<script type="module" src="/nexus/embed.js"></script>

<div style="height: 600px">
  <nexus-chat theme="auto"></nexus-chat>
</div>
```

- 元素填滿父容器，父容器要給高度。
- `theme`：`light`、`dark`、`auto`。
- `context`：情境標籤，例如畫面上選取的設備。可用屬性或 property 設，property 收字串或 `{ label }`，`null` 清掉。每送出一句話就附上當下的情境。
- 登入：開 serve 印的登入網址取得 cookie；宿主系統已有自己的登入時，改由後端換一次性登入票，交給 `await chat.signIn(ticket)`，快到期時元素會發 `nexus-ticket-needed`。

## 宿主頁工具：讓對話控制宿主頁

宿主頁宣告自己的工具給模型呼叫（例如轉鏡頭、高亮設備），模型叫了，元素發 `nexus-host-tool` 事件，宿主頁執行後交回結果。

```ts
const chat = document.querySelector('nexus-chat');
chat.hostTools = [
  {
    name: 'focus_device',
    description: '把鏡頭轉到指定的設備。',
    parameters: { type: 'object', properties: { id: { type: 'string' } }, required: ['id'] },
  },
];
chat.addEventListener('nexus-host-tool', (event) => {
  const { name, args, respond, fail } = event.detail;
  if (name !== 'focus_device') return;
  if (scene.focus(args.id)) respond({ focused: args.id });
  else fail(`找不到設備 ${args.id}`);
});
```

- `hostTools` 與監聽要在元素放進頁面之前、或同一個 task 裡設好。
- 每句話送出時附上當下的宣告，中途改的下一句才生效。
- `respond(結果)` 或 `fail(原因)` 只算第一次；期限預設 30 秒，逾時當失敗，這一輪照跑。
- 預設每次呼叫先出核准卡；哪幾支免核准由 serve 的 `host-tools` 設定決定，宿主頁自己標不算。
- 重新整理後沒交回的會以同一個 `id` 再發一次，不能重做的動作要用 `id` 記住做過沒有。
- 上限：至多 8 支，名字合 `^[a-z][a-z0-9_]{0,31}$`，`parameters` 是 `type: 'object'` 的 JSON Schema。

## 本機模擬模式：沒有 serve 也能開發

只拿到這包、沒有 harness 時，在 `web/` 跑 `pnpm run build:mock`，產物在 `web/dist-mock/`。宿主頁把 `embed.js` 換成那裡的 `embed-mock.js`（連同 `assets/` 放在同一個目錄）。元件照樣畫，但不打任何網路請求，照 `globalThis.nexusChatMock` 的劇本回話與叫宿主頁工具：

```ts
globalThis.nexusChatMock = {
  // 第 n 句話用第 n 輪；也可以改給 reply，每一句話自己決定。
  turns: [[{ say: '先轉過去。' }, { tool: 'focus_device', args: { id: 'B-07' } }, { say: '好了。' }]],
  reply: ({ text, context, hostTools }) => [{ say: `收到：${text}` }],
};
```

- 每一步是 `{ say }` 或 `{ tool, args, approval }`。叫工具預設先出核准卡（同 serve 出廠），`approval: false` 直接發 `nexus-host-tool`；`respond` 的結果、`fail` 的原因畫在工具卡上，再接下一步。
- 劇本每送出一句話讀一次，中途改了下一句生效。型別是 `nexus-chat.d.ts` 的 `NexusChatMock`。
- `web/embed-demo` 的測試頁網址加 `?embed=/embed-mock.js` 就是模擬模式，範例劇本在它的 `main.ts`：句子裡的設備編號轉過去，說到「標」再高亮。
- 驗得到的是宿主頁這一側的接線：宣告、事件、`respond`／`fail`、核准卡、`context`。**驗不到模型會不會在對的時機叫對的工具**：劇本不理解句子，要接真的 serve 才知道。
- 宿主頁 30 秒內沒交回，工具卡標失敗（「宿主頁沒在期限內交回」），劇本照跑下一步；核准卡沒有期限，等人按。
- 不模擬：斜線命令（清單是空的）、換票登入、上傳、子代理、對話清單（回「模擬模式不支援」）。`embed-mock.js` 只給本機開發，serve 不供應它。
