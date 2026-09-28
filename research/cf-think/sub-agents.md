# https://developers.cloudflare.com/agents/runtime/execution/sub-agents/ のまとめ

元記事: [Sub-agents](https://developers.cloudflare.com/agents/runtime/execution/sub-agents/)
Markdown版: [Sub-agents index.md](https://developers.cloudflare.com/agents/runtime/execution/sub-agents/index.md)
最終更新: 2026-09-15 (ドキュメント上の表記)
説明文: Spawn child agents with isolated storage and typed RPC using subAgent(), abortSubAgent(), and deleteSubAgent().

## 概要: Sub-agentsとは

- 子エージェントを同居するDurable Objectsとして生成し、それぞれ独自の分離されたSQLiteストレージを持ちます。
- 親は子メソッド呼び出し用の型付きRPCスタブを取得します。子クラスのすべての公開メソッドは、Promiseラップされた戻り値のリモートプロシージャコールとして呼び出せます。
- 単一ユーザーやエンティティが所有する、開放的な数の長期存続エージェントに向きます。例としてチャット、ドキュメント、セッション、シャード、プロジェクトが挙げられています。
- 各サブエージェントは独自状態で並列動作し、親は検出、アクセス制御、ライフサイクルを調整します。
- 親チャットエージェントがターン中に別チャットエージェントへ委譲し、子の進捗をインライン表示したい場合は、Agents as toolsを使います。Agents as toolsはsub-agents基盤の上に、親側runレジストリ、ストリーミングの `agent-tool-event` フレーム、リプレイ、キャンセル、クリーンアップを加えたものです。

## Quick start

- `Orchestrator` が `this.subAgent(Researcher, "research-1")` で子を取得し、`researcher.search(...)` を呼ぶ例が示されています。
- 両クラスはWorkerエントリーポイントからexportします。子専用クラスに個別のDurable Object bindingは不要で、`ctx.exports` 経由で自動検出されます。
- wrangler設定例ではトップレベル親 (`Orchestrator`) のみが `durable_objects.bindings` と `new_sqlite_classes` に登録されます。
- 子エージェントは親のfacetとして作成され、同じマシンを共有しますが、SQLiteストレージは完全分離です。

## subAgent

- 名前付きサブエージェントを取得または作成します。同名初回呼び出しで子の `onStart()` が発火し、以後は既存インスタンスが返ります。
- シグネチャは `subAgent<T extends Agent>(cls: SubAgentClass<T>, name: string): Promise<SubAgentStub<T>>` です。
- `cls` はエントリーポイントからexportされたAgentサブクラスで、export名とクラス名の一致が必要です。
- `name` は子インスタンスの一意名で、同名は常に同一子を返します。
- 戻り値は `SubAgentStub<T>` で、子定義の全公開インスタンスメソッドをPromise返却のリモート呼び出しとして公開します。

## SubAgentStub

- スタブが公開するのは子クラスで定義した公開インスタンスメソッドです。`Agent` 継承のライフサイクルフック、`setState`、`broadcast`、`sql` などは除外されます。
- 戻り値は自動でPromise化されます。同期値を返す `greet(name: string): string` は `Promise<string>` として扱われ、既に非同期の `fetchData` はそのままです。

## 要件

- 子クラスは `Agent` を継承します。
- 子クラスはWorkerエントリーポイントからexportします (`export class MyChild extends Agent`)。
- export名とクラス名の一致が必要で、`export { Foo as Bar }` 形は非対応です。
- トップレベル親クラスは `wrangler.jsonc` でDurable Object namespaceとしてbindします。
- facet専用子クラスは、他所でトップレベルDOとしてもbindしない限り `new_sqlite_classes` 登録不要です。
- ネストしたfacet親もトップレベルbinding不要で、ランタイムがルート親namespace経由で解決します。
- 子クラス名に `Sub` は使えません。`/sub/` がネスト経路の予約分離子です。

## テスト時の注意

- `@cloudflare/vitest-plugin` を使うテストでは、facetクラスをテスト専用DO bindingとして列挙し、`ctx.exports` がfacet互換のクラス値を提供できるようにする必要がある場合があります。
- その追加bindingはテスト用 `wrangler.jsonc` のみの話で、`new_sqlite_classes` には入れず、本番Worker要件ではありません。

## abortSubAgent

- 実行中サブエージェントを強制停止します。子は即時実行停止し、次回 `subAgent()` で再起動します。ストレージは保持され、実行インスタンスのみ破棄されます。
- シグネチャは `abortSubAgent(cls: SubAgentClass, name: string, reason?: unknown): void` です。
- `reason` は保留中または将来のRPC呼び出し側へ投げられるエラーです。
- 推移的で、子が独自サブエージェントを持つ場合も合わせて停止します。

## deleteSubAgent

- 子を停止 (実行中なら) した上でストレージを恒久削除します。次回 `subAgent()` は空SQLiteの新規インスタンスを作ります。
- シグネチャは `deleteSubAgent(cls: SubAgentClass, name: string): void` です。
- 削除も推移的で、子の子孫も合わせて削除されます。

## hasSubAgent

- 子が生成済みかつ未削除かを確認します。フレームワーク管理のSQLiteレジストリが裏付けです。
- 例として `if (!this.hasSubAgent(Chat, id)) return new Response("Not found", { status: 404 })` が示されています。

## listSubAgents

- 生成済みサブエージェントを列挙し、クラス指定での絞り込みが可能です。行は生成順で返ります。
- 例として `this.listSubAgents(Chat)` が `[{ className: "Chat", name: "chat-abc", createdAt: ... }]` 形を返すことが示されています。

## onBeforeSubAgent

- 親側で上書きするミドルウェアフックで、フレームワークが子を起こす前に `/sub/` 要求をゲート、改変、短絡できます。`onBeforeConnect` や `onBeforeRequest` に対応します。
- 戻り値の効果は、voidが元要求転送、Requestが改変転送、Responseが子を起こさない短絡です。
- 例として `Inbox` が `hasSubAgent(className, name)` で登録済みのみ許可し、未登録は404応答する厳密レジストリゲートが示されています。
- WebSocket upgrade要求も通常HTTP要求と同様にこのフックを流れます。改変Requestを返す場合は元WebSocket upgradeヘッダーを保持します。

## 親子identity

- 子は `this.parentPath` と `this.selfPath` で親子関係を知ります。
- `Inbox` 生成の `Chat` 内では、`parentPath` が `[{ className: "Inbox", name: "user-123" }]`、`selfPath` が親要素に自身 (`{ className: "Chat", name: "chat-abc" }`) を加えた配列になります。
- `parentPath` はルート優先順で、直親は常に `parentPath.at(-1)` です。トップレベルは `parentPath === []` です。
- 子から `parentAgent(Cls)` で直親への型付きRPCスタブを取得できます。例として `this.parentAgent(Inbox)` で `inbox.recordTurn(...)` する形が示されています。
- `parentAgent()` は直親がfacet限定サブエージェントでもルート側RPCブリッジで解決するため、ネスト親クラス群のトップレベルbindなしに直親呼び出しができます。
- 祖父母以上の先祖は `this.parentPath` を辿り `getAgentByName()` を直接使います。binding名とクラス名が異なる場合は `getAgentByName(env.MY_BINDING, this.parentPath.at(-1)!.name)` 形を使います。

## useAgent({ sub })

- `useAgent` 呼び出しに `sub` チェーンを足して子孫facetへ接続します。
- 例として `useAgent({ agent: "Inbox", name: userId, sub: [{ agent: "Chat", name: chatId }] })` が示されています。
- フックは `/agents/inbox/user-123/sub/chat/chat-abc` 形のURLを作り、`Chat` 子へ直接WebSocketを開きます。
- 他の `useAgent` 機能は通常通りで、state同期、`stub` 呼び出し、`@callable` RPC、返却ソケット上の `useAgentChat` が動作します。

## 直接HTTP・WebSocket URL

- `buildAgentPath()` でAgent identityの正規パスを作ります。同一パスがHTTP要求とWebSocket接続の両方に対応します。
- 例として `[{ className: "Inbox", name: userId }, { className: "Chat", name: chatId }]` と `{ leafPath: "/callbacks/job" }` から `/agents/inbox/{userId}/sub/chat/{chatId}/callbacks/job` を作ります。
- Agent内では `this.selfPath` を直接渡します。ルートDO binding名がクラス名と異なる場合は `rootBinding` オプションも渡します。
- `buildAgentUrl()` で公開originを足し、コールバック、webhook、承認、非同期ジョブ完了用URLを作ります。例として `Chat` が `this.env.PUBLIC_ORIGIN` と `this.selfPath` から `/callbacks/job` URLを作り、`onRequest` で同パスを処理します。
- 受信要求は `routeAgentRequest()` へ渡します。各祖先は宛先到達前に `onBeforeSubAgent` を実行します。子宛てではネスト `/sub/` 部分が除去され、子側パス名は `leafPath` 接尾辞になります。
- `buildAgentUrl()` のoriginはHTTP(S)またはWS(S)で、資格情報、パス、クエリ、フラグメントを含められません。コールバック用クエリは返却URLの `searchParams` で付けます。
- ルートAgent名は事前に有効なパス成分である必要があります。`sub` 成分は経路接頭辞、クラス名・binding名、ルートAgent名で予約されています。ヘルパーは子孫名をURLエンコードし、空白、Unicode、`/`、URL予約文字を含められます。

## カスタムHTTP経路

- トップレベルURL解析を自前fetchで行う場合、`routeSubAgentRequest()` で解決済み親stubから子へ振り分けます。
- 例として `/api/u/:userId/...` を正規表現で分け、`getAgentByName(env.Inbox, userId)` の親に `routeSubAgentRequest(request, parent, { fromPath: rest })` する形が示されています。
- `fromPath` は `/sub/chat/chat-abc` のような子テールを含むパスを受けます。`buildAgentPath()` 結果を直接渡せます。ヘルパーが解析し、親 `onBeforeSubAgent` を実行してfacetへ転送します。

## 外部からの型付きRPC

- 親DO内では `this.subAgent(Cls, name)` が型付きスタブを返します。親外からは `getSubAgentByName()` を使います。
- 例として `getAgentByName(env.Inbox, userId)` の親から `getSubAgentByName(inbox, Chat, chatId)` で子を取り、`chat.addMessage(...)` する形が示されています。
- `getSubAgentByName()` はRPC専用プロキシを返します。メソッド呼び出しは可能ですが `.fetch()` は例外になります。HTTPとWebSocket転送には `routeSubAgentRequest()` を使います。

## ストレージ分離

- 各サブエージェントは独自SQLiteを持ち、親や他子から完全分離されます。親の `this.sql` と子の `this.sql` は別データベースを操作します。
- 例として親が `parent_data` へ書き込み、子 `increment("clicks")` が子側 `counters` テーブルを操作し、両者が分離される形が示されています。

## 名前とidentity

- 異なるクラスは同一ユーザー向け名を共有でき、内部キーはクラス名とfacet名の複合で独立解決されます。
- 例として `subAgent(Counter, "shared-name")` と `subAgent(Logger, "shared-name")` は別ストレージの別子になります。
- 子の `this.name` はfacet名を返し、親名ではありません。

## パターン: 並列サブエージェント

- 複数子を並行実行します。例として `queries.map((query, i) => subAgent(Researcher, \`research-${i}\`).search(query))` を `Promise.all` する形が示されています。

## パターン: ネストサブエージェント

- 子がさらに子を作り木になります。例として `Manager.delegate` が `TeamLead` 子を呼び、`TeamLead.assign` が `Worker` 子を呼び、`Worker.execute` が完了結果を返す形が示されています。

## パターン: コールバックストリーミング

- `RpcTarget` コールバックを渡し、子から親へ結果を流します。
- 例として親側 `StreamCollector extends RpcTarget` が `onChunk(text)` で蓄積し、子 `Streamer.generate(prompt, callback)` がチャンク列で `callback.onChunk(chunk)` する形が示されています。

## スケジュールとdurable作業

- 子での動作表として以下が示されています。
  - `schedule()` / `scheduleEvery()` は子内で通常動作し、子内でコールバック実行します。
  - `cancelSchedule()` は呼び出し子所有分に動作します。
  - `getScheduleById()` / `listSchedules()` は呼び出し子スコープで動作します。
  - `keepAlive()` / `keepAliveWhile()` はトップ親へハートビート委譲で動作します。
  - `runFiber()` は動作し、fiber行とスナップショットは子SQLiteに保存されます。
  - `setState()` は子独自ストレージへ通常動作します。
  - `this.sql` は子独自SQLiteを指し通常動作します。
  - `subAgent()` は動作し、子がさらに子を作れます。
- 物理DOアラームはトップ親が所有し、facetは独立アラーム枠を持ちません。SDKが各予約コールバックや回復確認の所有子を記録し、親を起こして子へ振り分けます。コールバックは子を `this` として実行されるため、子状態、子SQLite、`getCurrentAgent()` 文脈を使います。
- 旧来同期的 `getSchedule()` と `getSchedules()` は予約行がトップ親保存のため子内で例外になります。`getScheduleById()` と `listSchedules()` を使います。
- 子内 `this.destroy()` は親へクリーンアップ委譲します。親は当該子の予約取消、子孫含む回復メタ削除、レジストリ削除、子ストレージ削除を要求します。子削除でisolate停止が戻り前に起き得るためfire-and-forget扱いです。

## 子からのWorkflows

- 子は `this.runWorkflow()` でWorkflowsを開始できます。追跡は子SQLite局所で、`AgentWorkflow.agent` はRPC、コールバック、状態更新、broadcastを発信子へ戻します。親は子開始workflowを自動列挙・制御しません。
- `SubAgentStub<T>` は子定義メソッドのみ公開するため、`getWorkflow()`、`approveWorkflow()`、`terminateWorkflow()` などの制御は子ラッパーメソッドを追加し、`await this.subAgent(Child, name)` 経由で呼びます。
- 子から `runWorkflow(..., { agentBinding })` を渡す場合、子binding名ではなくルートAgent binding名を使います。
- 子発信workflowの `AgentWorkflow.agent` はRPC専用です。Agentメソッド呼び出しに使い、外部HTTP・WebSocket経路には `this.agent.fetch()` ではなく `routeSubAgentRequest()` やネスト `/agents/{parent}/{name}/sub/{child}/{name}` 形URLを使います。

## 例

- [Multi-session chat example](https://github.com/cloudflare/agents/tree/main/examples/multi-ai-chat) が挙げられています。各チャットを分離状態と直接クライアント経路を持つAIChatAgentサブエージェントにした受信箱を作る例です。

## 関連

- [Think](https://developers.cloudflare.com/agents/harnesses/think/) はサブエージェント経由でAIターンを流す `chat()` メソッドの説明です。
- [Long-running agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/) は複数週間のエージェント存続文脈での子委譲の説明です。
- [Callable methods](https://developers.cloudflare.com/agents/runtime/lifecycle/callable-methods/) は `@callable` とサービスbinding経由RPCの説明です。
- [Agents as tools](https://developers.cloudflare.com/agents/runtime/execution/agent-tools/) はThinkまたは `AIChatAgent` 子を保持型ストリーミングツールとして動かす説明です。
- [Schedule tasks](https://developers.cloudflare.com/agents/runtime/execution/schedule-tasks/) はトップレベルと子の予約プリミティブの説明です。
