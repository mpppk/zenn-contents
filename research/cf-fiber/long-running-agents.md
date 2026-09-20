# Cloudflare Agents の Long-running agents 要約

参照元: [Long-running agents — Cloudflare Agents docs](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/)

調査日: 2026-09-18

公式ページの最終更新日は 2026-08-20 です。Cloudflare Agents SDK で、数日から数か月にわたって動作する Agent をどのように設計するかが、Project Manager Agent の例を通して説明されています。

## 概要

長期間動作する Agent は、常時稼働するプロセスとして実装するものではありません。永続的な identity、状態、スケジュール、回復情報を持つエンティティとして存在し、必要なイベントが発生したときに起動して処理します。

ページの要点は次の通りです。

- Agent は常時稼働するプロセスではなく、永続的な identity です。
- 状態、SQL データ、スケジュール、Fiber のチェックポイントは hibernation や再起動後も利用できます。
- メモリ上の変数、タイマー、実行中の `fetch`、ローカルクロージャは eviction 後に残りません。
- 数分の処理には `keepAlive()`、回復が必要な処理には `runFiber()`、呼び出し側に受付結果や状態確認が必要なジョブには `startFiber()` を使います。
- 大規模な多段処理には Workflows、複数の長期コンテキストを管理する場合には sub-agent を使います。

## Why Cloudflare for long-running agents

Agent は、ユーザー入力、LLM の応答、ツールの結果、人間による承認、スケジュールされた起動を待っている時間が長くなります。VM やコンテナでは待機中もサーバーの計算資源を確保しますが、Durable Object を基盤にした Agent は、hibernation 中に compute を消費しません。

Agent はアドレス可能なエンティティとして存在し、イベントが到着すると起動し、SQLite から状態を読み出し、処理後に再び休止します。これは actor model として説明されています。

ページでは、VM / コンテナとの違いを次のように整理しています。

| 観点 | VM / コンテナ | Durable Object |
| --- | --- | --- |
| アイドル時のコスト | 常に計算コストが発生 | hibernation 中は compute が発生しない |
| スケーリング | 容量を事前に確保・管理 | Agent 単位で自動的に扱う |
| 状態 | 外部データベースを用意する | SQLite を内蔵する |
| 回復 | プロセスマネージャーやヘルスチェックを構築する | プラットフォームの再起動後も状態を利用できる |
| identity / routing | ロードバランサーなどを構築する | 名前から Agent へルーティングする |

たとえば 10,000 個の Agent がそれぞれ 1% の時間だけ動作する場合、常時稼働する構成では 10,000 インスタンスを維持しますが、Durable Object では同時に動作するのは平均して約 100 インスタンスという説明です。

参考: [Why Cloudflare for long-running agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#why-cloudflare-for-long-running-agents)

## The lifecycle of a long-running agent

長期稼働 Agent は、連続して実行されるプロセスではなく、連続して存在しながら断続的に実行されるエンティティです。基本的なライフサイクルは次の通りです。

```text
Wake → onStart() → イベント処理 → idle（約 2 分）→ hibernation
  ↑                                              |
  └──────────── Alarm またはリクエストで再起動 ──┘
```

クラッシュや再デプロイによる eviction は、どの段階でも発生する可能性があります。状態は SQLite に保存され、次のイベントで Agent が再起動します。

参考: [The lifecycle of a long-running agent](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#the-lifecycle-of-a-long-running-agent)

## What survives

次の情報は、Agent の activation をまたいで保持されます。

- `setState()` の呼び出しごとに SQLite へ保存される `this.state`。
- Agent が作成した SQLite テーブルのデータ。
- SQLite に保存され、Alarm を起動するスケジュール。
- WebSocket クライアントごとの `connection.setState()` の状態。
- `runFiber()` の `stash()` で保存したチェックポイント。
- `startFiber()` が保持する managed fiber の状態行。

これらの SQLite 上のデータを基盤にした高レベル機能も、同じ永続性を利用できます。

参考: [What survives](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#what-survives)

## What does not survive

次の情報は、hibernation や eviction 後には保持されません。

- `setState()` や `this.sql` に保存していないメモリ上の変数やクラスフィールド。
- `setTimeout()` や `setInterval()` で動作中のタイマー。
- 実行中の `fetch` リクエスト。
- ローカルクロージャや Promise chain。

重要な処理は、状態として永続化するか、再開できる形で設計する必要があります。Agents SDK では、スケジュール、Fiber、Queue などがそのためのプリミティブになります。

参考: [What does not survive](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#what-does-not-survive)

## Running example: a project manager agent

ページ全体を通して、数週間から数か月にわたってプロジェクトを管理する `ProjectManager` Agent を例にしています。この Agent は次の処理を担当します。

- プロジェクトのタスクを管理する。
- sub-agent に仕事を割り当てる。
- 期限を確認してリマインダーを送る。
- GitHub の Webhook やチームメンバーからのメールに反応する。
- CI、コードレビュー、デプロイなどの長い処理を管理する。
- 再起動や eviction が何度発生しても、進行状態を保つ。

`ProjectState` にはプロジェクト名、ステータス、タスク一覧、計画を保持し、各タスクには担当、期限、状態、外部ジョブ ID などを持たせます。以降のセクションでは、この Agent に機能を追加する形で説明が進みます。

参考: [Running example: a project manager agent](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#running-example-a-project-manager-agent)

## Waking up: how agents get activated

hibernation 中の Agent は、外部イベントによって起動します。ページでは次の起動源を挙げています。

| 起動源 | 動作 | 例 |
| --- | --- | --- |
| HTTP request | Agent の URL へのリクエストが `onRequest()` を起動する | GitHub Webhook |
| WebSocket connection | クライアント接続時に `onConnect()` を起動する | ダッシュボードを開く |
| RPC call | Worker や別の Agent から RPC を呼び出す | Coordinator Agent からタスクを委譲する |
| Scheduled alarm | 保存済みのスケジュールが callback を起動する | 毎朝のリマインダー |
| Email | 受信メールが `onEmail()` を起動する | ステータスメールへの返信 |

Worker に届くイベントであれば、テレフォニーの Webhook やチャットボットなどにも同じパターンを適用できます。各イベントは Agent の名前を routing key として、同じ Durable Object インスタンスへ届けられます。イベントごとに Agent を個別起動・デプロイする必要はありません。

参考: [Waking up: how agents get activated](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#waking-up-how-agents-get-activated)

## Staying alive during long work

LLM のストリーミング、複数ツールの連続実行、遅い API の待機などがアイドル eviction の時間を超える場合、処理中に Agent が eviction される可能性があります。

`keepAlive()` は heartbeat を作成して inactivity timer をリセットします。`keepAliveWhile()` を使うと、非同期関数の実行中だけ heartbeat を維持し、関数の正常終了または例外発生時に自動的に停止します。手動で `keepAlive()` を使う場合は、`finally` で disposer を呼び出します。

参考: [Staying alive during long work](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#staying-alive-during-long-work)

## When keepAlive is not enough

ページでは、処理時間と手段を次のように対応づけています。

| 処理時間・要件 | 手段 |
| --- | --- |
| 数秒 | 通常のリクエスト処理 |
| 数分 | `keepAlive()` / `keepAliveWhile()` |
| 数分、再送される受付を安全に扱う必要がある | `startFiber()` |
| 数分から数時間 | Workflows |
| 数時間から数日 | ジョブを開始して休止し、完了通知で再起動する非同期パターン |

`keepAlive()` は Agent を動かし続けるための機能であり、数時間・数日続く外部ジョブを Agent 内で待ち続けるための機能ではありません。長い処理はジョブ ID を永続化して hibernation し、Callback、Polling、Workflow の完了通知などで再開します。

参考: [When keepAlive is not enough](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#when-keepalive-is-not-enough)

## Surviving crashes: fibers and recovery

Agent はデプロイ、ランタイム再起動、リソース制限などにより、いつでも eviction される可能性があります。途中状態を保存していなければ、処理中の仕事は失われます。

`runFiber()` は実行中の仕事を SQLite に登録し、`stash()` で中間状態を保存します。Agent が eviction された場合、Fiber の行が残り、次の activation で `onFiberRecovered()` が呼ばれます。回復時は最後のチェックポイントから処理する設計にしますが、何を再実行・再開・補償するかはアプリケーションが決めます。これは自動リプレイではありません。

`startFiber()` は、回復そのものに加えて、durable acceptance を必要とする場合に使います。冪等性キー、保持される状態、状態確認、キャンセル、クリーンアップを備えており、Webhook の再送による重複処理を防ぐ用途に向いています。デフォルトでは受付完了時に戻り、`waitForCompletion: true` を指定した場合は終端状態まで待ちます。

### Testing recovery locally

`wrangler dev` では本番と同じ回復動作を確認できます。Fiber を開始したあとに `wrangler` プロセスを Ctrl-C または SIGKILL で停止し、同じ永続化ディレクトリで再起動します。リクエストが届けば `onStart()` 経由で、クライアント接続がなくても永続化された Alarm 経由で回復が実行されます。

参考:

- [Surviving crashes: fibers and recovery](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#surviving-crashes-fibers-and-recovery)
- [Testing recovery locally](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#testing-recovery-locally)

## Handling long async operations

CI パイプライン、デザインレビュー、動画生成など、1 回の activation より長い処理は Agent が接続を保持したまま待たない設計にします。

基本パターンは次の通りです。

1. Agent が外部ジョブを開始します。
2. 外部ジョブの ID を Agent の状態に保存します。
3. Agent は hibernation します。
4. Callback、Polling、Workflow の完了通知で Agent が再起動します。
5. 保存していたジョブ ID と結果を照合し、状態を更新します。

この設計では、外部処理の待機時間に Agent の実行時間を消費しません。

参考: [Handling long async operations](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#handling-long-async-operations)

## Pattern: webhook callback

外部サービスが Callback に対応している場合のパターンです。Project Manager は CI パイプラインを開始し、自分の Callback URL とタスク ID を外部サービスへ渡します。パイプライン ID を Agent のタスクへ保存したあと、接続を保持せずに処理を終えます。

後から CI サービスが Callback URL へ結果を送ると、`onRequest()` が起動します。リクエストに含まれるタスク ID と結果を使って、タスクを `complete` または `blocked` へ更新します。

長い処理を開始するリクエストと、完了結果を受け取るリクエストを分離できる点が要点です。実運用では、Callback の署名検証、重複通知への対応、タスク ID と外部ジョブ ID の対応付けを追加で設計する必要があります。

参考: [Pattern: webhook callback](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#pattern-webhook-callback)

## Pattern: polling with schedule

外部サービスが Callback に対応していない場合は、Agent のスケジュールで状態を確認します。

動画生成 API へジョブを送信し、返されたジョブ ID とタスクを `in_progress` に更新したあと、`schedule()` で 60 秒後の Polling callback を登録します。Polling callback が完了または失敗を検出した場合はタスクを更新し、未完了なら次回の Polling を再スケジュールします。

サンプルでは、試行回数に応じて次の待機時間を増やし、最大 600 秒まで延長しています。Polling の間は Agent を常時稼働させず、スケジュールされた時刻だけ起動します。

参考: [Pattern: polling with schedule](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#pattern-polling-with-schedule)

## Pattern: workflow delegation

本番デプロイのように、build、test、stage、promote の各ステップを独立してリトライしたい処理は、Project Manager 自身で管理せず Workflow へ委譲します。

Agent は `runWorkflow()` で Workflow を開始し、返された instance ID をタスクに保存します。Workflow が処理を完了すると `onWorkflowComplete()` が呼ばれ、instance ID に対応するタスクを完了状態へ更新します。

Agent はプロジェクトの状態や外部イベントを管理し、ステップ実行とステップ単位のリトライは Workflow が管理するという分担です。

参考: [Pattern: workflow delegation](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#pattern-workflow-delegation)

## Reconstructing context after a long wait

外部ジョブが完了して Agent が起動しても、LLM が判断していた次のタスク、ステータス報告、ブロッカーへの対応などの思考コンテキストはメモリから失われています。長期稼働 AI Agent では、結果を受け取ったあとに推論をどのように再開するかが課題になります。

ページでは、次の三つの方法を示しています。

1. 会話履歴全体を再生する。`AIChatAgent` のメッセージを読み出し、結果を追加して LLM を再度呼び出します。実装は単純ですが、長い履歴を再処理します。
2. 継続用の要約を `stash()` する。「何を待っているか」「成功時と失敗時に何をするか」「関連するタスク ID や計画上の位置」を保存し、復帰時に短いプロンプトを作ります。
3. 構造化された計画をコンテキストとして使います。現在のステップ、完了済みの内容、次に行う処理を計画から読み出します。

公式ページでは、三番目の計画ベースの方法を、回復機構とコンテキスト再構築を兼ねる方法として説明しています。

参考: [Reconstructing context after a long wait](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#reconstructing-context-after-a-long-wait)

## Planning as a durability strategy

構造化された計画は、ユーザーへ進捗を表示するだけでなく、Agent の回復に使う永続データになります。

サンプルの `Plan` は、目標、ステップ一覧、現在のステップ番号、作成日時、更新日時を持ちます。各 `PlanStep` は ID、説明、状態、結果を持ち、状態は `pending`、`in_progress`、`complete`、`failed`、`skipped` で表します。

計画を使う処理の流れは次の通りです。

1. LLM で目標を具体的なステップへ分解します。
2. 計画を `setState()` で保存します。
3. `schedule(0, "executeNextStep")` で最初のステップを実行します。
4. ステップが完了したら結果と状態を保存し、現在のステップ番号を進めます。
5. 次のステップをスケジュールします。
6. 失敗した場合はステップを `failed` にして計画を保存します。

この方式では、再起動後に `currentStep` を調べて続きから再開できます。また、進捗をクライアントへ表示でき、失敗したステップ以降を再計画でき、計画を人間の承認ポイントとして利用できます。計画には LLM が現在位置、発生した結果、次の処理を把握するための情報も含まれます。

参考: [Planning as a durability strategy](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#planning-as-a-durability-strategy)

## Delegating to sub-agents

Project Manager がすべての処理を担うのではなく、専門的な処理を sub-agent へ委譲します。sub-agent はそれぞれ独自の identity、state、schedule、durable fiber、lifecycle を持ち、親の下に配置されます。

子 Agent の SQLite は親と分離され、callback の `this` も子 Agent になります。一方、sub-agent ごとに独立した物理 Alarm slot があるわけではありません。トップレベルの親が Alarm を所有し、どの子がスケジュールや Fiber の回復を所有しているかを記録して、起動した処理を子へルーティングします。

親は子が処理している間ずっとアクティブである必要はありません。子が処理を開始したあと、親は hibernation し、子のスケジュールまたは回復チェックで必要になったときに起動できます。

参考: [Delegating to sub-agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#delegating-to-sub-agents)

## Recovering interrupted LLM streams

LLM のストリーミングは、Agent が eviction されると途中で切断されます。`AIChatAgent` や `Think` では、各チャットターンを `runFiber()` で包み、ストリーミング中の `keepAlive` と、再起動後の回復フックを自動化しています。

`onChatRecovery()` では、eviction 前に生成された部分テキスト、`stash()` した回復情報、会話履歴、ターンの開始時刻などを使って回復方法を決めます。

ページで示されているプロバイダーごとの例は次の通りです。

| プロバイダー | 回復方法 | 特徴 |
| --- | --- | --- |
| Workers AI | 部分テキストから続行 | `continueLastTurn()` と assistant prefill を使う。トークンコストは低い |
| OpenAI Responses API | 完了済みレスポンスを取得 | ストリーミング中に `responseId` を保存し、回復時に取得する。追加トークンを使わない |
| Anthropic | 合成した継続要求 | 部分テキストを保存し、続行を求めるユーザーメッセージを生成する。トークンコストは中程度 |
| その他 | prefill を試し、未対応なら合成継続 | プロバイダーの機能に応じて切り替える |

古いターンを意図せず再開しないために、`ctx.createdAt` で回復対象が古すぎないかを判定できます。`AIChatAgent` と `Think` は durable recovery を常に使い、`chatRecovery` の設定で最大試行回数、終端メッセージ、回復失敗時の処理を調整できます。

Assistant のチャンクが一つも保存される前に中断された場合は、部分回答を続行できません。その場合、未回答のユーザーメッセージが最新状態なら、デフォルトではターン全体を再試行します。

参考: [Recovering interrupted LLM streams](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#recovering-interrupted-llm-streams)

## Managing state over time

数か月動作する Agent では、会話履歴、タイムライン、完了タスク、スケジュール記録が蓄積します。管理しなければデータが無制限に増加するため、定期的な整理が必要です。

参考: [Managing state over time](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#managing-state-over-time)

### Housekeeping

定期的なスケジュールで古いデータを整理し、完了済みの情報をアーカイブします。サンプルでは、`onStart()` で冪等な日次の `housekeeping` を登録し、30 日より前に完了したタスクを別の `archived_tasks` テーブルへ移します。また、完了またはエラー状態になってから 7 日以上経過した Workflow を削除します。

この処理により、現在の `state` と Workflow の保持量を抑えつつ、必要な履歴をアーカイブへ残します。

参考: [Housekeeping](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#housekeeping)

### Conversation history management

`AIChatAgent` を長期利用すると会話履歴が大きくなり、数か月分の履歴が LLM のコンテキストウィンドウを消費します。ページでは次の整理方法を示しています。

- Sliding window: 直近 N 件だけをアクティブなコンテキストへ渡す。
- Summarization: 古いメッセージを要約し、短い内容へ置き換える。監査用に元メッセージを SQLite に残すこともできます。
- Selective retention: 意思決定、承認、重要なコンテキストは残し、定型的なやり取りを削除する。

参考: [Conversation history management](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#conversation-history-management)

## End of life

プロジェクトの完了、調査の終了、監視期間の終了など、Agent の役割が終わった場合は明示的に終了処理を行います。

サンプルでは、すべてのスケジュールを取得してキャンセルし、状態を `complete` に更新したあと、`this.destroy()` を呼びます。`destroy()` は Agent の SQLite データ、スケジュール、状態を永久に削除する操作です。

後からデータが必要になる場合は、削除前に R2、D1、外部 API などへアーカイブします。再開する可能性がある Agent は destroy せず、完了状態にして hibernation させる方法もあります。アイドル状態の Agent は compute を消費しないためです。

参考: [End of life](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#end-of-life)

## When to use Workflows vs agent-internal patterns

Schedules、Fibers、Queues などの Agent 内部プリミティブと Workflows は、どちらも長期処理に使えます。使い分けは処理の性質で決めます。

| 観点 | Agent 内部の機能 | Workflows |
| --- | --- | --- |
| 適した処理 | Agent 自身のスケジュール、Polling、状態更新 | 独立した多段パイプライン |
| 耐久性 | SQLite に保存し、eviction 後も状態を使う | Workflow engine が処理を保持する |
| リトライ | `this.retry()` やスケジュール単位のリトライ | ステップ単位のバックオフ付きリトライ |
| 最大期間 | `keepAlive` を使った 1 回の activation は数分程度 | 1 ステップ 30 分、ステップ数に制限なし |
| 人間の承認 | state と WebSocket などを自分で実装する | `waitForApproval()` を利用する |
| 複雑さ | Agent 内で完結するため低い | 別クラスと Wrangler 設定が必要で高い |

実用上のルールとして、期限確認、状態同期、リマインダーなど Agent 自身のライフサイクル管理には schedules と Fibers を使います。デプロイ、データ処理、レポート生成など、独立して失敗・リトライできるパイプラインには Workflow を使います。

参考: [When to use Workflows vs agent-internal patterns](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#when-to-use-workflows-vs-agent-internal-patterns)

## Summary

Cloudflare の長期稼働 Agent は、常時実行されるプロセスではなく、数週間から数か月にわたって存在し、必要なときだけ起動・処理・休止する durable entity です。

ページでは、主なプリミティブを次のように整理しています。

| プリミティブ | 目的 |
| --- | --- |
| `setState()` / `this.sql` | activation 間で状態を保持する |
| `schedule()` / `scheduleEvery()` | 将来の時刻に Agent を起動する |
| `keepAlive()` / `keepAliveWhile()` | アクティブな長時間処理中の eviction を避ける |
| `runFiber()` / `stash()` | 長い処理をチェックポイントし、回復する |
| `startFiber()` | ジョブを durable に受け付け、確認・キャンセルする |
| `chatRecovery` | 中断された LLM ストリームを回復する |
| `onRequest()` / `onEmail()` / RPC | 外部イベントで Agent を起動する |
| `runWorkflow()` | 重い多段処理を Workflow へ委譲する |
| `subAgent()` | 専門処理を子 Agent へ委譲する |
| state 内の構造化された計画 | 回復、進捗表示、再計画に使う |

Project Manager Agent は、目標を計画へ分解し、ステップを実行し、Webhook・メール・スケジュールへ反応し、チェックポイントから回復し、sub-agent と Workflow へ委譲し、古いデータを整理し、完了時に自分自身を終了します。

この一連の処理に、Agent が連続稼働する必要はありません。必要なのは、次のイベントで同じ identity と永続状態から処理を再開できることです。

参考: [Summary](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#summary)

## Related

ページから参照されている関連トピックは次の通りです。

- [Durable Execution](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/): `runFiber()`、`startFiber()`、`stash()`、クラッシュ後の回復。
- [Schedule tasks](https://developers.cloudflare.com/agents/runtime/execution/schedule-tasks/): 遅延、cron、interval による起動。
- [Retries](https://developers.cloudflare.com/agents/runtime/execution/retries/): リトライの設定とパターン。
- [Workflows](https://developers.cloudflare.com/agents/runtime/execution/run-workflows/): durable な多段処理。
- [Store and sync state](https://developers.cloudflare.com/agents/runtime/lifecycle/state/): `setState()` と状態の永続化。
- [WebSockets](https://developers.cloudflare.com/agents/runtime/communication/websockets/): lifecycle hook と hibernation。
- [Callable methods](https://developers.cloudflare.com/agents/runtime/lifecycle/callable-methods/): `@callable` や service binding による RPC。
- [Email](https://developers.cloudflare.com/agents/communication-channels/email/): 受信メール。
- [Webhooks](https://developers.cloudflare.com/agents/communication-channels/webhooks/): 外部イベント。
- [Human in the loop](https://developers.cloudflare.com/agents/concepts/human-in-the-loop/): 承認フロー。

## 調査上のポイント

- Long-running Agent は「長い処理を 1 回のリクエストで保持する」設計ではなく、「イベントの間で状態を保存し、必要なときに再起動する」設計です。
- `keepAlive()` は処理を延命する API、Fiber は処理を回復する API、Workflow は独立した多段処理を管理する API と整理できます。
- 長い外部ジョブでは、Fiber で接続を保持するよりも、ジョブ ID を保存して hibernation し、Callback または Polling で再開する設計が基本です。
- LLM Agent では処理状態だけでなく、次に何をするかを示す計画や継続要約も永続化する必要があります。
- 長期稼働では、回復だけでなく、データの整理、会話履歴の圧縮、明示的な終了処理まで設計対象になります。
