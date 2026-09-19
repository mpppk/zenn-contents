# Cloudflare Agents SDK の Fiber 調査メモ

調査日: 2026-09-18

## 調査対象

Cloudflare Agents SDK における Fiber と durable execution を調査する。起点は [Durable execution with fibers](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/) である。

ここでいう Fiber は、Durable Object 上で実行する処理を SQLite に記録し、途中状態をチェックポイントし、Durable Object の eviction や再起動後に回復処理へ渡すための実行プリミティブである。記事では、一般的なコンピュータサイエンスにおける軽量スレッドとしての Fiber ではなく、Cloudflare Agents SDK の `runFiber()` / `startFiber()` を扱う。

公式ドキュメントの [Durable execution with fibers](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/) は 2026-08-20 更新である。

## 先に押さえる結論

- `runFiber()` は、Agent 自身の処理を durable に実行する API である。実行開始前に SQLite へ回復用の行を登録し、実行中は `keepAlive()` を内部で保持する。
- `ctx.stash(data)` または `this.stash(data)` で、次の回復に必要な JSON シリアライズ可能な状態を SQLite に保存する。`stash()` は同期的に書き込まれ、毎回前回のスナップショット全体を置き換える。
- Durable Object が途中で eviction されると、元の async 関数やクロージャは失われる。次の activation で `onFiberRecovered(ctx)` が呼ばれ、`name` と最後の `snapshot` をもとに、再実行・続行・補償・通知などの方針をアプリケーションが決める。
- `runFiber()` の復旧は自動リプレイではなく、回復フックにドメイン固有の処理を書く仕組みである。自動リトライもない。
- `startFiber()` は、バックグラウンド処理を永続的に受け付け、すぐ応答し、冪等性キーで重複受付を抑え、後から状態確認・キャンセルするための managed fiber API である。
- `keepAlive()` は eviction の可能性を下げるだけで、eviction 後の回復を提供しない。回復が必要な処理には `runFiber()` または `startFiber()` を使う。
- Agent 内部の仕事には Fiber、独立した多段処理・ステップ単位のリトライ・長い待機・承認待ちには Workflows を使う、という使い分けが公式ドキュメントで示されている。

## なぜ Fiber が必要か

Agents SDK の Agent は Durable Object を基盤としている。Durable Object は常時稼働するプロセスではなく、イベントを処理していないときは hibernation できる。状態や SQL データは永続化されるが、メモリ上の変数、タイマー、実行中の `fetch`、ローカルクロージャは eviction 後に残らない。

Durable Object の eviction の原因として公式ドキュメントは次を挙げている。

1. リクエストや WebSocket 接続がない状態での非アクティブタイムアウト。目安は約 70〜140 秒。
2. コード更新やランタイム再起動。発生頻度は非決定的で、1 日に 1〜2 回起きる場合があると説明されている。
3. Alarm handler の 15 分のタイムアウト。

途中で eviction が発生すると、LLM プロバイダー、外部 API、データベースなどへの HTTP 接続が切断される。ストリーミングバッファ、途中結果、ループカウンタ、複数ターンの処理位置もメモリから失われる。

参考:

- [Durable execution with fibers — Why fibers exist](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#why-fibers-exist)
- [Long-running agents — What survives / What does not survive](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#what-survives)

## `keepAlive()` と Fiber の違い

`keepAlive()` は 30 秒間隔の heartbeat を作り、非アクティブタイムアウトによる eviction を避けるための API である。`keepAliveWhile(fn)` は関数の開始前に heartbeat を始め、関数が終了または throw したあとに停止する。手動で `keepAlive()` を使う場合は disposer を呼ぶ必要がある。

公式ドキュメントは、次のように使い分けている。

- 遅い API の応答を数分待つだけなら `keepAlive()`。
- 中間結果を持つ多段計算なら `runFiber()`。
- 10 分以上かかるバックグラウンド調査ループなら `runFiber()` と `stash()`。
- Webhook の仕事を重複なく受け付けたいなら `startFiber()`。

`keepAlive()` は、コード更新、Alarm handler のタイムアウト、リソース制限などによる eviction を回復可能にはしない。`runFiber()` は `keepAlive()` を内部で使い、SQLite に回復情報を保存する。

参考: [Durable execution with fibers — keepAlive](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#keepalive)

## `runFiber()` の基本モデル

基本形は次のようになる。

```ts
import { Agent } from "agents";
import type { FiberRecoveryContext } from "agents";

export class ResearchAgent extends Agent {
  async doWork() {
    await this.runFiber("research", async (ctx) => {
      const step1 = await expensiveOperation();
      ctx.stash({ step1 });

      const step2 = await anotherExpensiveOperation(step1);
      this.setState({ ...this.state, result: step2 });
    });
  }

  async onFiberRecovered(ctx: FiberRecoveryContext) {
    if (ctx.name !== "research") return;

    const snapshot = ctx.snapshot as { step1: unknown } | null;
    if (snapshot) {
      const step2 = await anotherExpensiveOperation(snapshot.step1);
      this.setState({ ...this.state, result: step2 });
    }
  }
}
```

処理の流れは次の通りである。

1. `runFiber(name, fn)` が回復用メタデータを SQLite に保存する。
2. `keepAlive()` の heartbeat が開始される。
3. `fn(ctx)` が実行される。
4. `ctx.stash(data)` を呼ぶたびに、次の回復に使うスナップショットが SQLite に保存される。
5. 正常終了すると回復用の行が削除され、`fn` の戻り値が呼び出し元へ返る。
6. `fn` がエラーを投げた場合も回復用の行は削除され、エラーが呼び出し元へ伝播する。通常の実行エラーに対する自動リトライはない。

Durable Object が途中で eviction された場合は、処理中の行が残る。次の activation でリクエストが届いたとき、または永続化された Alarm が発火したときに未完了の Fiber が検出され、`onFiberRecovered()` が呼び出される。バックグラウンド Agent にクライアントからの接続がない場合でも、永続化された Alarm が回復処理を起こす経路になる。

### `FiberContext`

`runFiber()` の callback は `FiberContext` を受け取る。公式リファレンスに記載された主な値は次の通りである。

- `id`: Fiber の ID。
- `signal`: キャンセルを受け取る `AbortSignal`。
- `stash(data)`: 現在の回復スナップショットを SQLite に保存する関数。
- `snapshot`: Fiber の開始時点で保持されていたスナップショット。通常は `null` から始まる。

`this.stash(data)` も同じ処理を行う。`ctx` をネストした関数へ渡したくない場合に使えるが、`runFiber()` の callback 外で呼ぶと例外になる。

### `stash()` の性質

- `data` は JSON シリアライズ可能である必要がある。
- `stash()` は同期的に SQLite へ書き込む。呼び出しが戻った後に eviction されても、そのデータは保存済みである。
- 1 回の `stash()` は前回のスナップショットへマージされず、スナップショット全体を置き換える。回復に必要な情報を毎回すべて書く必要がある。
- チェックポイントの粒度はアプリケーションが決める。一般には外部 API 呼び出しや副作用の前後で、完了済みステップと未完了ステップを保存する。

参考: [Durable execution with fibers — Checkpoints with stash](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#checkpoints-with-stash)

## eviction 後の回復

Fiber の callback はシリアライズされない。回復時に元の lambda を再構成することはできず、`onFiberRecovered()` が受け取れるのは `name`、`snapshot`、Fiber の識別情報などである。そのため回復ロジックは callback の中だけに閉じず、回復フックにも明示的に実装する。

回復時の方針はフレームワークが固定しない。実装で選べる方針は次の通りである。

- 最初から再実行する。
- `stash()` のスナップショットから未完了ステップだけ続行する。
- 既に発生した外部副作用を補償する。
- ユーザーへ中断を通知して処理を終える。
- 何もせず、状態を調査対象として残す。

`runFiber()` の行は `onFiberRecovered()` が正常終了すると削除される。回復フックの中でさらに `runFiber()` を呼ぶ場合は、新しい Fiber の行が作られる。回復フックが throw すると行が残り、後の起動または Alarm による再スキャンで回復が試みられる。

回復フックが常に失敗する場合、回復 Alarm は指数バックオフし、遅延は最大 5 分になる。`fiberRecoveryMaxAgeMs` のデフォルトは 24 時間で、その期間を超えると `fiber:recovery:skipped` イベントが記録されて破棄される。`fiberRecoveryMaxAgeMs: 0` にすると無期限に保持されるが、回復不能な行が存在する間は Durable Object が idle eviction されないため、用途を意識して設定する。

重要なのは、Fiber が exactly-once の外部副作用を自動的に保証する機能ではない点である。回復後にどこから再開するか、同じ Webhook や外部 API 呼び出しを重複させないか、補償処理を行うかはアプリケーション側で設計する。

参考: [Durable execution with fibers — Recovery](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#recovery)

## inline 実行と fire-and-forget

`runFiber()` は二つの呼び出し方に対応する。

```ts
// inline: 結果を待つ
const result = await this.runFiber("work", async () => {
  return computeExpensiveThing();
});

// fire-and-forget: 呼び出し元は待たない
void this.runFiber("background", async () => {
  await longRunningProcess();
});
```

inline で待っている間に Durable Object が eviction されると、元の caller も失われる。回復処理は `onFiberRecovered()` から起きるが、元の caller へ戻り値を返すことはできない。長時間処理で、呼び出し元が受付結果・状態・キャンセルを必要とする場合は `startFiber()` を使う。

## `startFiber()` の基本モデル

`startFiber()` は managed fiber を作成する。callback の実行結果を返す API ではなく、仕事を永続的に受け付けたことと managed fiber の状態を返す API である。

```ts
const receipt = await this.startFiber(
  "reply-to-webhook",
  async (ctx) => {
    ctx.stash({ webhookId, threadId });
    await postReply(threadId);
  },
  {
    idempotencyKey: `webhook:${webhookId}`,
    metadata: { threadId },
  },
);

if (!receipt.accepted) {
  // 同じ idempotencyKey の Webhook は以前に受付済み
}
```

主な性質は次の通りである。

- callback の実行前に retained fiber record が保存される。
- `fiberId`、現在の `status`、`metadata`、`accepted` を含む受付結果が返る。
- `idempotencyKey` が同じ重複呼び出しは、可能なら同じ isolate で実行中の処理へ合流し、それ以外は保持された状態を返す。重複呼び出しの `accepted` は `false` になる。
- デフォルトでは durable に受付できた時点で戻る。`waitForCompletion: true` を指定すると terminal status まで待つ。
- callback の戻り値は保存されない。後から `inspectFiber()` または `inspectFiberByKey()` で managed fiber の状態を確認する。
- `cancelFiber()` は `aborted` 状態を記録し、現在の isolate で実行中なら `ctx.signal` も abort する。callback は高コスト処理や可視の副作用の前に `ctx.signal.aborted` を確認する必要がある。キャンセルは協調的であり、callback が signal を確認しなければ処理そのものは止まらない。

managed fiber の状態として公式 API リファレンスには次が示されている。

`pending`、`running`、`completed`、`aborted`、`interrupted`、`error`

### managed fiber の操作 API

- `inspectFiber(fiberId)` / `inspectFiberByKey(idempotencyKey)`: 状態を 1 件取得する。
- `listFibers(options)`: `status`、`name`、`limit` で絞って一覧する。
- `cancelFiber(fiberId, reason)` / `cancelFiberByKey(idempotencyKey, reason)`: managed fiber を `aborted` にする。
- `resolveFiber(fiberId, result)`: `interrupted` 状態をアプリケーションの回復後に解決する。
- `deleteFibers(options)`: 完了・エラー・中断済みの保持行を削除する。デフォルトでは `interrupted` は削除対象にならず、調査または手動解決のために残る。

### callback の結果を取得する方法

`startFiber()` の callback は `Promise<void>` として扱われ、callback の戻り値は保存されない。結果が必要な場合は、callback 内で `ctx.stash()` に保存し、`StartFiberResult.snapshot` または `inspectFiberByKey()` の戻り値に含まれる `snapshot` から取得する。

同じリクエスト内で完了まで待つ場合は、`waitForCompletion: true` を指定する。

```ts
type CalculationResult = {
  value: number;
};

const fiber = await this.startFiber(
  "calculate",
  async (ctx) => {
    const value = await calculate();
    ctx.stash({ result: { value } satisfies CalculationResult });
  },
  {
    idempotencyKey: "calculate:001",
    waitForCompletion: true,
  },
);

if (fiber.status === "completed") {
  const snapshot = fiber.snapshot as { result: CalculationResult };
  return snapshot.result;
}
```

受付だけ先に完了させる場合は、`idempotencyKey` や `fiberId` を使って後から状態を取得する。

```ts
const fiber = await this.inspectFiberByKey("calculate:001");

if (fiber?.status === "completed") {
  const snapshot = fiber.snapshot as { result: CalculationResult };
  return snapshot.result;
}
```

`stash()` はチェックポイント全体を置き換えるため、結果以外に回復で必要な情報も残す場合は、同じ `snapshot` オブジェクトへまとめて保存する。大きな結果や業務データとして管理する結果は、`this.sql` や `setState()` に保存し、`fiberId` または `idempotencyKey` で参照する設計も使える。

## 回復の単位と冪等性

`runFiber()` の `name` は回復フックで処理種別を判定するための識別子であり、一意ではない。同じ名前の Fiber を複数実行できる。受付の重複排除が必要な場合に `name` をキーとして使うのではなく、`startFiber()` の `idempotencyKey` を使う。

外部副作用がある処理では、次の情報をチェックポイントに含める案がある。

- 処理対象の外部 ID。
- 完了済みステップ。
- 未完了ステップ。
- 外部 API に送った idempotency key。
- 既に投稿・送信・更新を行ったかを示す状態。
- 回復時に再実行してよい期限や試行回数。

これは Fiber API の自動機能ではなく、Fiber を使う側の設計事項である。

## 複数 Fiber と sub-agent

複数 Fiber を同時に実行できる。各 Fiber は独自の SQLite 行とスナップショットを持ち、各 Fiber が `keepAlive()` の参照を持つ。参照カウントにより、すべての Fiber が終わるまで Durable Object は heartbeat を維持する。回復時には孤立したすべての行について `onFiberRecovered()` が呼ばれるため、`ctx.name` などで処理種別を分岐する。

sub-agent でも Fiber を利用できる。Fiber の行とスナップショットは子の SQLite に保存され、`onFiberRecovered()` の `this` は子 Agent になる。一方で Durable Object の物理的な Alarm slot は sub-agent ごとには存在せず、トップレベルの親が持つ。親が回復メタデータを管理し、子へ処理をルーティングする。

参考: [Sub-agents — Scheduling and durable work](https://developers.cloudflare.com/agents/runtime/execution/sub-agents/#scheduling-and-durable-work)

## Workflows との使い分け

公式ドキュメントで示される比較を記事では次のように説明できる。

| 観点 | Fiber | Workflows |
| --- | --- | --- |
| 主な対象 | Agent 自身の処理 | Agent から切り離せる独立したジョブ |
| 状態・回復 | Agent の SQLite、`stash()`、`onFiberRecovered()` | Workflow エンジンの durable step |
| リトライ | 自動ではない。回復フックで方針を決める | ステップごとのリトライとバックオフを使う |
| 適した処理 | Agent の調査ループ、途中状態を持つ実行、Webhook の受付 | ビルド・テスト・デプロイ、データ処理、承認待ちを含む多段処理 |
| 呼び出し側 | `runFiber()` は戻り値を待てるが、eviction 後は元の caller が失われる | Workflow ID などで外部から追跡する |

`ThinkWorkflow` のドキュメントでは、Workflows は複数の deterministic step、長い待機、人間による承認を持つプロセス向けであり、`startFiber()` は Agent 内部で回復するアプリケーション所有の冪等なジョブ向けと整理されている。

参考:

- [Long-running agents — When to use Workflows vs agent-internal patterns](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/#when-to-use-workflows-vs-agent-internal-patterns)
- [Think Workflows](https://developers.cloudflare.com/agents/harnesses/think/workflows/)

## Chat Agent との関係

`AIChatAgent` と `Think` は Fiber を内部で利用し、各チャットターンを自動的に回復可能な Fiber で包む。LLM ストリーミングが途中で中断した場合、フレームワーク内部の回復処理が動作し、必要なら `onChatRecovery` でプロバイダー固有の回復方針を実装できる。

記事では、Fiber を単なる長時間処理 API として説明するだけでなく、Agents SDK のチャットターン回復の土台でもあると説明できる。ただし、チャットストリームの再接続による回復と、Durable Object eviction 後の回復は別の問題である。公式ドキュメントは、クライアント切断後の resumable streaming と、サーバー側の Durable Object eviction recovery を分けて説明している。

参考:

- [Durable execution with fibers — Chat recovery](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#chat-recovery)
- [Resumable Streaming — Cloudflare Agents](https://github.com/cloudflare/agents/blob/main/docs/agents/resumable-streaming.md)

## ローカルでの検証方法

`wrangler dev` では Fiber の回復が本番と同じように動くと説明されている。

1. Agent を起動し、`runFiber()` を実行する。
2. `wrangler` プロセスを Ctrl-C または SIGKILL で停止する。
3. 同じ永続化ディレクトリを使って再起動する。
4. リクエストが届けば `onStart()` 経由で、クライアント接続がなければ永続化された Alarm 経由で回復が実行される。

記事のサンプルでは、各ステップに短い遅延を入れ、`stash()` で `completed` と `pendingSteps` を保存したあとにプロセスを停止すると、最後のチェックポイントから再開する様子を確認できる。

参考: [Durable execution with fibers — Testing locally](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#testing-locally)

## 記事で扱えそうな主張

### 主張候補

Fiber は「実行を永久に動かし続けるスレッド」ではなく、「失われる可能性がある実行に、SQLite の回復記録とチェックポイントを付け、再起動後にアプリケーションの回復ロジックを呼び出す仕組み」である。

この表現なら、Fiber が Durable Object の hibernation を無効化するものではなく、`keepAlive()` による延命と eviction 後の recovery を組み合わせるものであることを説明できる。

### 記事の構成案の種

1. Durable Object 上の Agent は常時稼働プロセスではない
2. 途中で失われる処理をどう回復するか
3. `runFiber()`、`stash()`、`onFiberRecovered()` の最小例
4. `keepAlive()` だけでは回復できない理由
5. `startFiber()` による Webhook の durable acceptance と冪等性
6. 回復時に自動リプレイされない理由と外部副作用の扱い
7. Fiber と Workflows の境界
8. `wrangler dev` で eviction recovery を検証する

### 例に向く題材

- 3 ステップの調査処理。各ステップの結果を `stash()` に保存し、`pendingSteps` から再開する。
- Webhook の再送。`startFiber()` に `idempotencyKey` を渡し、同じ Webhook が二重受付されないことを示す。
- LLM ストリーミング。自前実装ではなく、`AIChatAgent` / `Think` が Fiber を内部利用している関係を補足する。

## Cosense の参照結果

リポジトリの指示に従い、`niki-ai`、`niki-auth`、`niki-tech`、`niki-cs` を確認した。

### niki-ai

検索で Fiber に該当したページは [🔖 Agents · Cloudflare Agents docs](https://scrapbox.io/niki-ai/%F0%9F%94%96Agents_%C2%B7_Cloudflare_Agents_docs) である。Cloudflare Agents SDK について、セッションごとの永続 identity とローカル SQL ストレージ、スケジュールタスク、durable execution（Fibers）、障害から復帰可能な実行、複数モデル対応などを整理している。

このページから確認できる記事上の補助線は、Fiber を単独の API としてではなく、Agents SDK の「状態・スケジュール・回復」を構成する要素として置くことである。

### niki-tech

関連するページとして [Cloudflare Thinkフレームワーク実装のポイント](https://scrapbox.io/niki-tech/Cloudflare_Think%E3%83%95%E3%83%AC%E3%83%BC%E3%83%A0%E3%83%AF%E3%83%BC%E3%82%AF%E5%AE%9F%E8%A3%85%E3%81%AE%E3%83%9D%E3%82%A4%E3%83%B3%E3%83%88) を確認した。Fiber という個別ページは検索で見つからなかったが、Think の実装を Durable Object として捉え、Slack Thread、Chat SDK Thread、Thread Agent を 1:1:1 の隔離単位として整理している。

Fiber 記事では、Think のような Agent 実装を対象にする場合でも、実処理を持つ Agent の identity・状態・実行回復が別々の概念であることを説明する材料になる。関連ページには [Cloudflare Workers](https://scrapbox.io/niki-tech/Cloudflare_Workers) や Durable Objects の概念ページへのリンクもある。

### niki-auth

`Fiber` で検索した範囲では該当ページは見つからなかった。認証や Agent の外部副作用を記事の中心にする場合は、別途このプロジェクトの関連ページを探す。

### niki-cs

`Fiber` で検索した範囲では該当ページは見つからなかった。現時点では Fiber 記事の中心資料として使える内容は確認できていない。

## 参照 URL

### Cloudflare 公式

- [Durable execution with fibers](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/)
- [Long-running agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/)
- [Sub-agents](https://developers.cloudflare.com/agents/runtime/execution/sub-agents/)
- [Think Workflows](https://developers.cloudflare.com/agents/harnesses/think/workflows/)
- [Cloudflare Agents](https://developers.cloudflare.com/agents/)
- [Project Think: building the next generation of AI agents on Cloudflare](https://blog.cloudflare.com/project-think/)

### Cloudflare の公式リポジトリ

- [cloudflare/agents: durable-execution.md](https://github.com/cloudflare/agents/blob/main/docs/agents/durable-execution.md)
- [cloudflare/agents: long-running-agents.md](https://github.com/cloudflare/agents/blob/main/docs/agents/long-running-agents.md)
- [cloudflare/agents: resumable-streaming.md](https://github.com/cloudflare/agents/blob/main/docs/agents/resumable-streaming.md)
- [cloudflare/agents: forever-fibers example](https://github.com/cloudflare/agents/tree/main/experimental/forever-fibers)

### Cosense

- [niki-ai](https://scrapbox.io/niki-ai/)
- [niki-auth](https://scrapbox.io/niki-auth/)
- [niki-tech](https://scrapbox.io/niki-tech/)
- [niki-cs](https://scrapbox.io/niki-cs/)

## 次の調査・執筆で確認すること

- `agents` パッケージの現行型定義で `StartFiberResult`、`FiberInspection`、`FiberRecoveryResult` のフィールドを確認する。
- `startFiber()` の重複受付時に返る status と、managed fiber の回復後に `onFiberRecovered()` が返す `FiberRecoveryResult` の型をサンプルで検証する。
- `stash()` に保存できない値、最大サイズ、回復行の保持上限など、公開ドキュメントに明記されていない制約をソースコードまたは実行テストで確認する。
- Workflows と Fiber の比較で、Workflows のリトライ・待機・承認の仕様を必要な範囲だけ追加確認する。
- 記事 phase では、外部 URL をサービス仕様の根拠としてインライン脚注にし、末尾の「参考」セクションにも列挙する。
