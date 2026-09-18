## Thinkとは何か
* Thinkはトップレベルエージェント(WebSocket経由で`useAgentChat`と通信)とサブエージェント(RPC経由で親エージェントが`chat()`で駆動)の両方として動作する [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* Cosenseの関連メモ: Cloudflare Agents SDKはWorkers基盤上で状態を持つAIエージェントを構築するSDKで、セッションごとのSQLストレージとdurable executionを持つ [参考](https://scrapbox.io/niki-ai/Cloudflare_Agents_SDK)

## Think発表時のブログ
* 2026-04-15のProject Think発表では、Agents SDKの次世代としてdurable execution・sub-agent・sandboxed code execution・persistent sessionを束ねたものとして紹介された [参考](https://blog.cloudflare.com/project-think/)
{TODO: ブログ内容のサマリを箇条書きで記載}

## AI Agent実装時に必要な要素
* `@cloudflare/think`はAgents SDK上のopinionatedなチャットエージェント基底クラスで、`getModel()`を実装するとagentic loop・永続化・streaming・ツール実行・stream resumption・extensionsが動作する [参考](https://developers.cloudflare.com/agents/harnesses/think/)
{TODO: ここで挙げられている要素それぞれの説明を箇条書きで記載}

## AIChatAgentとの違い
* どちらも`Agent`を継承し、同じ`cf_agent_chat_*` WebSocketプロトコルを使う [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* AIChatAgentはプロトコルアダプタで、`onChatMessage`内で`streamText`呼び出しやツール配線を利用者が書く。Thinkは`getModel()`・`getSystemPrompt()`または`configureSession()`・`getTools()`を上書きし、既定の`onChatMessage`がagentic loopを実行する [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* 最小サブクラスはAIChatAgentが約15行に対してThinkは3行(`getModel()`のみ) [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* ストレージはAIChatAgentがフラットなSQLテーブル、ThinkはSessionによるツリー構造メッセージ・context block・compaction・FTS5 [参考](https://developers.cloudflare.com/agents/harnesses/think/)

## 最小実装と配線
* `npm i @cloudflare/think @cloudflare/ai-chat agents ai @cloudflare/shell zod workers-ai-provider`で導入し、`Think`を継承して`getModel()`でWorkers AIモデルを返す。`routeAgentRequest`でWorker entryから振り分ける [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* Getting Startedでは`@cf/moonshotai/kimi-k2.6`を使った`MyAgent`の定義、`wrangler.jsonc`の`ai` binding・DO binding・migration設定、React側の`useAgent`+`useAgentChat`が手順化されている [参考](https://developers.cloudflare.com/agents/think/getting-started/)
* wrangler設定例は`compatibility_date`・`nodejs_compat`・`durable_objects.bindings`・`new_sqlite_classes`を含む [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* クライアントは既存の`useAgentChat`がそのまま使える [参考](https://blog.cloudflare.com/project-think/)

## Session・memory・context管理
* `configureSession`で`soul`(読み取り専用の identity)と`memory`(書き込み可能な事実メモリ)などのcontext blockを定義し、モデルが`set_context`ツールで更新する [参考](https://developers.cloudflare.com/agents/think/getting-started/)
* 会話履歴はツリー構造で、regenerationは旧応答を残したまま分岐する。compactionは古いメッセージの削除ではなくoverlayによる非破壊サマリ [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* FTS5によるセッション内・セッション横断の全文検索がある [参考](https://developers.cloudflare.com/agents/harnesses/think/)

## ツール統合
* 毎ターンでworkspaceツール・`getTools()`の独自ツール・extensionツール・sessionツール(`set_context`等)・skillツール・MCPツール・clientツールがマージされる [参考](https://developers.cloudflare.com/agents/think/getting-started/)
* 組み込みworkspaceファイルツール(read/write/edit/list/find/grep/delete)があり、`read`は行番号付きテキストと画像・PDFのマルチモーダル受け渡しに対応する [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* code executionはDynamic Workersと`@cloudflare/codemode`上の`createExecuteTool`で、workspace filesystem・browser(CDP)・任意ToolSetをconnectorとして束ね、実行はdurableに記録される [参考](https://github.com/cloudflare/agents/releases/tag/%40cloudflare%2Fthink%400.9.0)
* agentがTypeScriptでextensionを自作し、Dynamic Workerに読み込んで新ツールを登録できる。extensionはDOストレージに永続化される [参考](https://blog.cloudflare.com/project-think/)

## ターン起動API
* `runTurn(options)`が統一入口で、`wait`(結果を待つ)・`submit`(durableに受理して後で状態確認)・`stream`(RPC向けにコールバックへ流す)の3モードがある [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* 旧来のショートカットとして`saveMessages()`・`submitMessages()`・`chat()`・`continueLastTurn()`・`addMessages()`が残る。用途別の対応表がドキュメントにある [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* `submit`は`submissionId`や`idempotencyKey`による冪等な再送が可能で、webhookやRPC呼び出し向け [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* `addMessages()`は推論を起動せずtranscriptに追記するため、ツール`execute`内から呼んでもデッドロックしない [参考](https://developers.cloudflare.com/agents/harnesses/think/)

## 長時間実行・sub-agent・recovery
* chat turnはrecovery fiber内で実行され、DOがevictされても中断したターンを継続またはリトライする。応答は`accepted`・`streaming`・`completed`でスナップショットされる [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* `chat()`による親子間streaming RPC、`agentTool()`による委譲、`getScheduledTasks()`による定期ターン、`startFiber()`によるwebhook前後の冪等処理、Workflows連携がある [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* primitivesとしてfibersによるdurable execution・独立SQLiteを持つsub-agent・sandboxed code execution(execution ladder: workspace・isolate・npm・browser・sandbox)が単体利用可能 [参考](https://blog.cloudflare.com/project-think/)

## lifecycle・observability
* `beforeTurn`・`onChatResponse`などのhookが毎ターンで発火する [参考](https://developers.cloudflare.com/agents/think/getting-started/)
* 内部で`wrapAISDK()`による計装済みで、`observability.traces.enabled`を有効化すると`invoke_agent`・`chat`・`execute_tool`・`tool_approval`スパンがWorkers Observabilityに送られ、DashboardのAgents viewで確認できる [参考](https://developers.cloudflare.com/agents/harnesses/think/)
* モデルはWorkers AIのほか差し替え可能で、CosenseのAgents SDKメモでも複数モデル差し替え対応として整理されている [参考](https://scrapbox.io/niki-ai/Cloudflare_Agents_SDK)

## 注意点・experimental
* Project Thinkはexperimentalで、API surfaceは安定しているが変更の可能性がある [参考](https://blog.cloudflare.com/project-think/)
* 2026-08-18の`@cloudflare/think@0.16.0`ではconvention-drivenなThink framework層(Vite plugin・生成Worker entry・`think` CLI・Studio等)が削除され、hand-writtenなWorker entryによる明示的runtime利用に戻った。既存利用者は`agents/vite`・`routeAgentRequest`・`wrangler types`等への移行が必要 [参考](https://github.com/cloudflare/agents/releases/tag/%40cloudflare%2Fthink%400.16.0) [参考](https://github.com/cloudflare/agents/pull/1613)
* AI SDKはv6とv7を支援し、`ai`と`@ai-sdk/react`のメジャーを揃える必要がある [参考](https://developers.cloudflare.com/agents/harnesses/think/)
