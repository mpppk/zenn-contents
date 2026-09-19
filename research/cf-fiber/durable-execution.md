# Durable execution with fibers の要約

参照元: [Durable execution with fibers — Cloudflare Agents docs](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/)

調査日: 2026-09-18

公式ページの最終更新日は 2026-08-20 です。

## 概要

Cloudflare Agents SDK の Fiber は、Durable Object 上で実行中の処理を eviction 後も回復できるようにする実行単位です。

`runFiber()` は処理を SQLite に登録してから実行し、`stash()` で途中状態を保存します。Durable Object が処理途中で eviction された場合、次の activation で `onFiberRecovered()` が呼び出されます。

`startFiber()` は、バックグラウンド処理を永続的に受け付けるための API です。呼び出し元へ受付結果をすぐ返し、idempotency key による重複排除、状態確認、キャンセルを扱えます。

Fiber は処理を常時実行し続けるための仕組みではありません。実行中の処理に回復情報を付け、Durable Object の再起動後にアプリケーションの回復ロジックを呼び出す仕組みです。

## Quick start

基本的な `runFiber()` は、処理途中で `ctx.stash()` を呼び、`onFiberRecovered()` で保存済みの状態を読み取ります。

```ts
import { Agent } from "agents";
import type { FiberRecoveryContext } from "agents";

export class MyAgent extends Agent {
  async doWork() {
    await this.runFiber("my-task", async (ctx) => {
      const step1 = await expensiveOperation();
      ctx.stash({ step1 });

      const step2 = await anotherExpensiveOperation(step1);
      this.setState({ ...this.state, result: step2 });
    });
  }

  async onFiberRecovered(ctx: FiberRecoveryContext) {
    if (ctx.name !== "my-task") return;

    const snapshot = ctx.snapshot as { step1: unknown } | null;
    if (snapshot) {
      const step2 = await anotherExpensiveOperation(snapshot.step1);
      this.setState({ ...this.state, result: step2 });
    }
  }
}
```

この例では `step1` の完了後にスナップショットを保存します。途中で eviction された場合、回復処理は `step1` のスナップショットを使って `step2` を実行します。

## Fiber が必要になる理由

Durable Object は次の要因で eviction される可能性があります。

1. リクエストや WebSocket 接続がない状態での非アクティブタイムアウト。目安は約 70〜140 秒です。
2. コード更新やランタイム再起動。発生頻度は非決定的で、1 日に 1〜2 回起きる場合があります。
3. Alarm handler の 15 分のタイムアウトです。

eviction が処理途中で起きると、LLM プロバイダー、API、データベースなどへの接続が切断されます。また、ストリーミングバッファ、途中応答、ループカウンタなどのメモリ上の状態が失われます。

`keepAlive()` は非アクティブによる eviction の可能性を下げますが、コード更新、Alarm handler のタイムアウト、リソース制限などによる eviction 後の回復は行いません。`runFiber()` は実行情報を SQLite に保存し、回復フックへ処理を渡します。

参考: [Why fibers exist](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#why-fibers-exist)

## `keepAlive()`

`keepAlive()` は 30 秒間隔の heartbeat を作り、Durable Object の非アクティブタイムアウトを避けるための API です。

```ts
const dispose = await this.keepAlive();
try {
  await longWork();
} finally {
  dispose();
}
```

`keepAliveWhile()` を使うと、非同期関数の開始時に heartbeat を開始し、関数が終了または例外を投げた時点で停止します。

```ts
const result = await this.keepAliveWhile(async () => {
  return await slowApiCall();
});
```

`keepAlive()` の heartbeat は `listSchedules()` には表示されません。Agent が作成したスケジュールと Alarm slot を共有しますが、スケジュールの行は追加しません。

heartbeat の間隔は static options の `keepAliveIntervalMs` で変更できます。デフォルト値は 30 秒です。

### `keepAlive()` と `runFiber()` の使い分け

公式ドキュメントでは、次の例が示されています。

| 処理 | API |
| --- | --- |
| 遅い API の応答を待つ | `keepAlive()` |
| `AIChatAgent` の LLM ストリーミング | 内部で自動処理 |
| 中間結果を持つ多段計算 | `runFiber()` |
| 10 分以上かかるバックグラウンド調査 | `runFiber()` と `stash()` |
| Webhook の仕事を重複なく受け付ける | `startFiber()` |

参考: [keepAlive](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#keepalive)

## `runFiber()` の API

`runFiber()` の型は次のように定義されています。

```ts
runFiber<T>(
  name: string,
  fn: (ctx: FiberContext) => Promise<T>,
): Promise<T>
```

Fiber は callback を実行する前に SQLite へ回復用の行を登録します。その後、`keepAlive()` の heartbeat を保持しながら callback を実行します。

callback が正常終了すると、回復用の行が削除され、callback の戻り値が呼び出し元へ返ります。callback が例外を投げた場合も回復用の行は削除され、エラーが呼び出し元へ伝播します。通常の実行エラーに対する自動リトライはありません。

`name` は `onFiberRecovered()` で処理種別を判定するための識別子です。同じ名前の Fiber を複数実行できるため、`name` は一意な受付キーではありません。

### `FiberContext`

`runFiber()` の callback が受け取る `FiberContext` には、次の値があります。

- `id`: Fiber の ID です。
- `signal`: キャンセルを受け取る `AbortSignal` です。
- `stash(data)`: 現在の Fiber のスナップショットを保存します。
- `snapshot`: Fiber 開始時点で保持されていたスナップショットです。通常は `null` です。

## Fiber のライフサイクル

### 通常の実行

通常の `runFiber()` は次の順で進みます。

1. 回復用のメタデータを SQLite に保存します。
2. `keepAlive()` の heartbeat を開始します。
3. callback を実行します。
4. `ctx.stash(data)` が呼ばれた場合、スナップショットを保存します。
5. callback が終了すると回復用メタデータを削除します。
6. heartbeat を停止し、callback の戻り値を返します。

### eviction 後の回復

Durable Object が eviction されると、メモリ上の callback とローカル状態は失われます。SQLite に保存された回復用の行は残ります。

次の activation では、リクエストや接続による `onStart()`、または永続化された Alarm によるスキャンによって中断された Fiber が検出されます。その後、Fiber ごとに `onFiberRecovered(ctx)` が呼ばれます。

クライアント接続がないバックグラウンド Agent では、永続化された Alarm が回復処理を起動する経路になります。

### Sub-agent での実行

Sub-agent の Fiber も利用できます。Fiber の行とスナップショットは子 Agent 自身の SQLite に保存され、`onFiberRecovered()` は子 Agent を `this` として実行されます。

Sub-agent ごとに独立した物理 Alarm slot があるわけではありません。トップレベルの親が物理的な Alarm を持ち、回復チェックを該当する子 Agent へルーティングします。

### callback のエラー

`runFiber()` の callback が通常のエラーを投げた場合、回復用の行は削除され、エラーが呼び出し元へ伝播します。Fiber の回復が必要なエラー処理は `onFiberRecovered()` に実装します。

参考: [Lifecycle](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#lifecycle)

## inline と fire-and-forget

`runFiber()` は、呼び出し元が結果を待つ inline 形式と、呼び出し元が結果を待たない fire-and-forget 形式に対応します。

```ts
// inline
const result = await this.runFiber("work", async () => {
  return computeExpensiveThing();
});

// fire-and-forget
void this.runFiber("background", async () => {
  await longRunningProcess();
});
```

inline 形式で待機中に Durable Object が eviction された場合、元の呼び出し元も失われます。回復時に `onFiberRecovered()` は呼び出されますが、元の呼び出し元へ戻り値を返すことはできません。

呼び出し元が受付結果、状態確認、キャンセルを必要とする長時間処理には `startFiber()` を使います。

## `startFiber()`

`startFiber()` は、バックグラウンド処理を durable に受け付けるための API です。callback を実行する前に retained fiber record を保存し、その後に callback をバックグラウンドで実行します。

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
  // 同じ idempotencyKey の処理が受付済み
}
```

主な特徴は次の通りです。

- `fiberId`、現在の `status`、`metadata`、`accepted` を含む受付結果を返します。
- `idempotencyKey` を指定すると、同じキーを使った重複呼び出しを判定できます。重複呼び出しでは `accepted` が `false` になります。
- デフォルトでは、処理が durable に受け付けられた時点で戻ります。
- `waitForCompletion: true` を指定すると、managed fiber が terminal status になるまで待ちます。
- callback の戻り値は保存されません。`startFiber()` は値を返す API ではなく、処理の受付と状態を返す API です。

managed fiber の状態には、`pending`、`running`、`completed`、`aborted`、`interrupted`、`error` があります。

## managed fiber の操作

`startFiber()` で作成した managed fiber には、次の操作 API があります。

| API | 内容 |
| --- | --- |
| `inspectFiber(fiberId)` | Fiber ID で状態を取得します |
| `inspectFiberByKey(idempotencyKey)` | idempotency key で状態を取得します |
| `listFibers(options)` | `status`、`name`、`limit` で一覧します |
| `cancelFiber(fiberId, reason)` | Fiber を `aborted` にします |
| `cancelFiberByKey(idempotencyKey, reason)` | idempotency key で Fiber をキャンセルします |
| `resolveFiber(fiberId, result)` | `interrupted` 状態を回復後の結果で解決します |
| `deleteFibers(options)` | 保持された managed fiber の行を削除します |

キャンセルは協調的に行われます。現在の isolate で実行中の callback には `ctx.signal` の abort が伝わりますが、callback 自身が `ctx.signal.aborted` を確認しなければ処理は止まりません。高コスト処理や外部へ見える副作用の前に signal を確認します。

`deleteFibers()` はデフォルトでは `completed`、`error`、`aborted` の行を削除対象にします。`interrupted` の行は、調査や手動の回復に使えるよう、デフォルトでは削除対象になりません。

## `stash()` によるチェックポイント

`ctx.stash(data)` は Fiber の状態を SQLite に同期的に書き込みます。`stash()` の呼び出しが戻った後に eviction が発生しても、保存したデータは SQLite に残ります。

```ts
await this.runFiber("research", async (ctx) => {
  const steps = ["search", "analyze", "synthesize"];
  const completed: string[] = [];
  const results: Record<string, unknown> = {};

  for (const step of steps) {
    results[step] = await executeStep(step);
    completed.push(step);

    ctx.stash({
      completed,
      results,
      pendingSteps: steps.slice(completed.length),
    });
  }
});
```

`stash()` は前回のスナップショットへマージされません。毎回、回復に必要な状態全体を保存します。保存するデータは JSON シリアライズ可能である必要があります。

`this.stash(data)` も同じ処理を行います。ネストした関数から `ctx` を受け渡さずに保存したい場合に使えますが、`runFiber()` の callback 外で呼ぶと例外になります。

## Recovery の実装

Fiber の callback はシリアライズされません。回復時に利用できるのは `name`、スナップショット、Fiber の識別情報などです。そのため、元の callback を自動的に再生するのではなく、`onFiberRecovered()` に回復処理を実装します。

回復処理では、次のような方針を選べます。

- 最初から処理を再実行します。
- 最後のチェックポイントから未完了のステップだけを続行します。
- 外部副作用に対する補償処理を行います。
- ユーザーへ中断を通知します。
- 処理を続行せず、調査対象として状態を残します。

`runFiber()` の行は `onFiberRecovered()` が正常終了すると削除されます。回復フック内で `runFiber()` をもう一度呼ぶと、新しい Fiber として処理が開始されます。

回復フックが例外を投げた場合は行が残り、後の起動または Alarm によるスキャンで再度回復が試みられます。再試行には指数バックオフが使われ、遅延は最大 5 分です。

回復フックが常に失敗する場合の保持期間は `fiberRecoveryMaxAgeMs` で制御されます。デフォルトは 24 時間で、期間を超えると `fiber:recovery:skipped` イベントが記録されて破棄されます。`fiberRecoveryMaxAgeMs: 0` を指定すると無期限に保持されますが、回復不能な行が存在する間は Durable Object が idle eviction されません。

`startFiber()` の managed fiber では、`onFiberRecovered()` から `FiberRecoveryResult` を返して、`completed`、`error`、`aborted`、`interrupted` のいずれかへ状態を遷移させます。`undefined` を返すと `interrupted` のまま保持されます。

参考: [Recovery](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#recovery)

## Chat recovery

`AIChatAgent` と `Think` は Fiber を内部で利用します。各チャットターンが回復可能な Fiber に包まれ、LLM ストリーミングが途中で中断した場合の回復処理が行われます。

プロバイダーごとの回復方針を実装する場合は `onChatRecovery` を使います。通常の Fiber と同様に、回復時の再実行によって外部副作用が重複しないかを考慮する必要があります。

## Concurrent fibers

複数の Fiber を同時に実行できます。各 Fiber は独自の SQLite 行とスナップショットを持ち、個別に `keepAlive()` の参照を保持します。すべての Fiber が完了するまで Durable Object は heartbeat を維持します。

回復時には、孤立したすべての Fiber について `onFiberRecovered()` が呼び出されます。`ctx.name` などを使って処理種別を判定します。

## ローカルでのテスト

`wrangler dev` では、本番と同じように Fiber recovery をテストできます。

1. Agent を起動して `runFiber()` を実行します。
2. `wrangler` プロセスを Ctrl-C または SIGKILL で停止します。
3. 同じ永続化ディレクトリを使って `wrangler` を再起動します。
4. リクエストが届くと `onStart()` 経由で回復し、クライアント接続がない場合は永続化された Alarm 経由で回復します。

ローカルの SQLite と Alarm の状態はプロセス再起動後も保持されます。

参考: [Testing locally](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/#testing-locally)

## このページから読み取れる API の使い分け

| 状況 | API |
| --- | --- |
| 処理が短く、回復用のチェックポイントが不要 | 通常の Agent メソッド |
| 数分かかる処理中に非アクティブ eviction を避ける | `keepAlive()` / `keepAliveWhile()` |
| Agent 内部の多段処理を途中から回復する | `runFiber()` と `stash()` |
| 呼び出し元へすぐ応答し、後から状態を確認する | `startFiber()` |
| Webhook の再送を同じ仕事として扱う | `startFiber()` と `idempotencyKey` |
| Agent から独立した多段処理、ステップ単位のリトライ、長い待機 | Workflows |

## 関連ページ

- [Long-running agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/): Fiber、スケジュール、Agent の状態を組み合わせて、数日から数か月にわたる Agent を設計する方法です。
- [Schedule tasks](https://developers.cloudflare.com/agents/runtime/execution/schedule-tasks/): Alarm を使って Agent を将来起動する方法です。
- [Sub-agents](https://developers.cloudflare.com/agents/runtime/execution/sub-agents/): 子 Agent 内で Fiber やスケジュールを使う場合の動作です。
- [Run Workflows](https://developers.cloudflare.com/agents/runtime/execution/run-workflows/): Agent から Workflows を起動する方法です。
- [Chat agents](https://developers.cloudflare.com/agents/communication-channels/chat/chat-agents/): `AIChatAgent` のチャットと Fiber recovery の関係です。

## 要約

Cloudflare Agents SDK の Fiber は、Durable Object の eviction によって失われる可能性がある処理を、SQLite の回復用レコードとチェックポイントで扱う機能です。

`runFiber()` は Agent 内部の処理を実行し、`stash()` で途中状態を保存し、`onFiberRecovered()` で回復方針を実装します。`startFiber()` は処理を durable に受け付け、idempotency key、状態確認、キャンセルを提供します。

Fiber は callback の自動リプレイや外部副作用の exactly-once 実行を提供しません。処理の再開位置、重複実行の防止、補償処理、回復期限は Agent 側で設計します。
