## Durable Object(以下DO)とは何か
* ユニークなIDごとにインスタンスが動的生成される、特殊なCloudflare Worker
* それぞれのインスタンスが専用のストレージを持つことで、強整合性と高速アクセスを両立する
* 他のCloudflare Workerから参照することで、複数のWorkerから共通のin-memory Stateを参照可能
* Actorモデルとも言える
* 一定時間利用しないと自動的にアイドル状態になり、課金が発生しなくなる

## 動作サンプル(カウンター)

* {複数のworkerから共通のDOを参照し、カウンターを操作できることがわかる図}

```typescript
import { DurableObject } from "cloudflare:workers";

export interface Env {
  COUNTERS: DurableObjectNamespace<Counter>;
}

export class Counter extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);

    this.ctx.storage.sql.exec(`
      CREATE TABLE IF NOT EXISTS counter (
        id INTEGER PRIMARY KEY,
        value INTEGER NOT NULL
      );

      INSERT OR IGNORE INTO counter (id, value)
      VALUES (1, 0);
    `);
  }

  async increment(): Promise<number> {
    const row = this.ctx.storage.sql
      .exec<{ value: number }>(`
        UPDATE counter
        SET value = value + 1
        WHERE id = 1
        RETURNING value
      `)
      .one();

    return row.value;
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const name =
      new URL(request.url).searchParams.get("name") ?? "default";

    const stub = env.COUNTERS.getByName(name);
    const value = await stub.increment();

    return Response.json({ name, value });
  },
};
```

* `env.COUNTERS>getByName(name)`で参照すると、動的にDOが生成される
* DO生成時に、Counterクラスのコンストラクタが実行される
  * DOはアクセスがない場合、自動的にアイドル状態に遷移する。アイドル状態から復帰する場合は再度コンストラクタが実行される
* Counterクラスで定義した処理が、DOのインスタンス上で実行される
  * この例では`stub.increment()`が呼び出されているが、その処理は呼び出し元のworkerではなくDOのインスタンス上で実行される
* DOはSQLiteベースのストレージを持っており、この例ではctx.storage.sqlでアクセスしている
  * このストレージはインスタンスごとに分離されている
  * Tips: 共通のDBが欲しい場合はCloudflare D1を使用する

## FAQ

### DOはどのようなライフサイクルを持つ?

以下を参照。

![Durable Object Lifecycle](https://developers.cloudflare.com/cdn-cgi/image/onerror=redirect,width=2866,height=2276,format=webp/_astro/durable-object-lifecycle.DdQka9ef.png)
from [durable-object-lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle)

* ポイント
  * リクエストが来ると、DOはActiveになる
  * リクエスト処理が完了すると、一定時間はメモリ上のデータを保持したままIdle状態になる
  * Idle状態のまま一定時間が経過するとHibernatedやInactiveになり、メモリ上のデータが解放される
  * 状態遷移をフックして処理を実行することはできないため、失われて困るメモリ上のデータは先に永続化しておく必要がある

### ORMは利用できる?

できる。基本的にはただのSQLiteなので、`ctx.storage.sql.exec`経由でSQLを発行するようなドライバがあれば良い。

* [Drizzleの公式ドライバ](https://orm.drizzle.team/docs/get-started/do-existing)
* [kysely-cloudflare(コミュニティ製)](https://www.npmjs.com/package/%40oselvar/kysely-cloudflare)

### D1との違い

* D1に対してDOはよりprimitiveな概念。実際に、D1のデータベースは1つのDOで構築される。
* D1は純粋なストレージであるのに対し、DOはWokerによる処理系を組み合わせた概念。
  * オンメモリでデータを保持することで高速にデータを呼び出し元へ返却することが可能
  * DOの処理系とストレージが同じマシンに置かれることが保証されている
* D1はデータベース単位で静的に生成される。DOはID単位で動的に生成される。
* D1はread replicaの作成が可能

### WebSockets Hibernation

* DOを停止(Hibernate)しつつ、WebSocketの接続は維持する仕組み
* Hibernatedな状態で新しくメッセージが届くと、DOがActiveに復帰する
* DOを停止させられるので、メッセージのやり取りのない間は課金が行われないというメリットがある
* DO停止中はCloudflareのネットワークが接続を維持してくれる

from [Use WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/#durable-objects-hibernation-websocket-api)

### 料金

課金はcompute + storageの2軸。PaidはWorkers Paidの最低$5/moに含まれる。

#### compute

| 項目 | Free | Paid | 備考 |
| --- | --- | --- | --- |
| Requests | 10万件/日 | 月100万件まで無料、超過$0.15/百万件 | HTTP・RPCセッション・WSメッセージ・alarmを含む |
| Duration (wall-clock × 128MB換算) | 1.3万GB-s/日 | 月40万GB-sまで無料、超過$12.50/百万GB-s | `Active` + `Idle non-hibernateable`のみ課金 |

* 通常Workers(Paid)との比較
  * リクエスト料金は通常Workersに対してDOは半額
  * duration課金は通常Workersは無し。代わりにCPU timeで課金

#### storage

| 項目 | Free | Paid | 備考 |
| --- | --- | --- | --- |
| Rows read | 500万/日 | 月250億まで無料、超過$0.001/百万行 | `get`等KV互換APIも行換算 |
| Rows written | 10万/日 | 月5000万まで無料、超過$1.00/百万行 | `setAlarm`・deleteも1行扱い |
| Stored data | 5GB | 5GB-monthまで無料、超過$0.20/GB-month | 空DB約12KBも課金対象 |

* D1と同一料金

* from [Durable Objects Pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/) / [Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
