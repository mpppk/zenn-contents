## Fiberとは

* CloudflareのAgentはDO上で実行される。DOの処理の中断が発生しても途中から利用可能な、再開可能な処理を定義、制御する仕組みがFiber
* Fiber自体が新しい機能を提供するわけではない。DOが提供するストレージやAlarmなどのプリミティブを基盤とした抽象概念

 ## DOの処理の中断はいつ、なぜ起きるか
* Inactivity timeout — ~70–140 seconds with no incoming requests or open WebSockets
  * 例えばLLMのAPIから長時間ストリームでメッセージを受け取っている状況が該当する
* Code updates / runtime restarts — non-deterministic, 1–2x per day
* Alarm handler timeout — 15 minutes
* (TODO: この3つを日本語にする)

この処理の中断の対策として、Fiberは以下の2つのアプローチを提供する
* keepAlive: 処理の中断をpreventする
* runFiber: 処理が中断されても途中から再実行できるようにする

## keepAlive

```ts
const dispose = await this.keepAlive();
try {
	await longWork();
} finally {
	dispose();
}
```
* disposeするまでDOはevictされない
* 内部ではAlarmを利用し、30秒ごとにheartbeatを送信している

## runFiber

```ts
import { Agent } from "agents";
import type { FiberRecoveryContext } from "agents";

class MyAgent extends Agent {
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
* runFiberに渡した関数の実行中はDOがevictされない
  * 内部ではkeepAliveを呼び出している
* Fiberの処理が中断された場合、Agentが自動的にDOを再起動し、onFiberRecoveredを呼び出す。この時DOのメモリ上の状態は失われているため、onFiberRecoveredはsnapshotから状態を復元する責務を持つ

## startFiber

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
	// This webhook was already accepted by an earlier delivery.
}
```

* runFiberがfiberに渡した関数の戻り値を返すのに対し、startFiberはfiberの実行状況を返す
* また、idempotentKeyを設定することにより、同じキーに対して呼び出された場合でもexactly onceな実行を保証する

## AIChatAgentやThinkでのFiberの利用例
* AIChatAgent と Think は、チャットターンを内部的にFiberとして実行する
* LLMからのメッセージストリーミングが切断された際にonChatRecoveryが呼び出され、この中で回復処理を行う。