---
title: "AIエージェントのharnessを比較する — ベンチマークのスコアは「どの足場で測ったか」に依存する"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "llm", "aiagent", "claudecode", "opencode"]
published: false
---

コーディングエージェントの harness（Claude Code、Codex CLI、OpenCode、Cursor Agent…）が増えすぎて、何を基準に選べばいいのか分からなくなっている人向けに、選定の軸を3つに整理する。

先に結論を書いておく。**harness の選定は固定的な優劣では決まらない。**「いまのモデルで何を補う必要があるか」で決まり、モデルが変われば答えも変わる。そして harness は選定の対象であると同時に、**モデルのベンチマークスコアがどの harness で測られたかという測定条件でもある** — こちらは後半で扱う。

## harness とは何で、なぜ比較するのか

エージェントは `agent = model + harness` と分解できる。モデル単体はトークンを予測するだけで、ファイルを読むことも、コマンドを実行することも、失敗をやり直すこともできない。それを与えるのが harness — loop、tools、context 管理、memory、permissions、sandbox、検証ループといった足場の総体である。

harness に単一の完成形はない。同じモデルを、異なる前提（loop・tools・context 管理・権限・UI）で包んだ実装が並立している。以下、3 つの軸で横断する。

## 軸1: 提供主体と実行場所

| 分類 | 例 | 性格 |
| --- | --- | --- |
| プラットフォーム公式CLI | Claude Code (Anthropic) / Codex CLI (OpenAI) / Gemini CLI (Google) / Qwen Code (Alibaba) | モデル提供元が直接配布。そのモデルで最も検証されており、ベンチマークの実行環境にも使われる（後述） |
| OSS CLI | OpenCode (SST) / OpenClaw / Hermes Agent (Nous Research) / Pi / Aider / Goose (Block) | プロバイダ非依存。モデルを切り替えられる |
| IDE統合 | Cursor Agent | エディタプロセス内で完結し、権限モデルと UI が CLI 型と異なる |
| ランタイム/ラッパー | deepseek-harness / ori harness (OpenRouter) | 前者はプラグインの動的差し替え、後者は請求・ガードレールの一元化が主目的 |

この分類が効くのは、**「何を差し替えたいか」で層が決まる**からである。モデルを差し替えたいなら OSS CLI、認証と請求を差し替えたいならラッパー、実行環境そのものを組み替えたいならランタイムを見る。公式 CLI はどれも差し替えられない代わりに、そのモデルで最も検証されている。

## 軸2: 設定と可搬性

harness ごとに設定ファイルが違う。これが乗り換えの摩擦になる。

| harness | 設定 |
| --- | --- |
| Claude Code | `CLAUDE.md` + `.claude/settings.json` |
| Codex CLI | `AGENTS.md` + `codex` 設定 |
| OpenCode | `opencode.json` + `AGENTS.md` |
| OpenClaw | `AGENTS.md` 自動注入 + `openclaw agents add --workspace` |
| Hermes Agent | プロファイルごとの `config.yaml` と、エージェントの人格を定義する `SOUL.md` |
| Pi | RPC（`pi --mode rpc`、JSON Lines）|

`AGENTS.md` が事実上の共通形式に寄りつつある一方、権限や memory の置き場所は harness 固有のままで、ここは移植できない。

（表の Pi の行だけ粒度が違う。ファイルで設定するのではなく、外部プロセスから駆動する界面そのものが設定にあたる、という設計だからである。この節の最後で触れる）

この摩擦を吸収しようとするのが、`.agent/` のような共通フォルダに memory と skills を置いて複数 harness から読ませる「可搬 brain」の試みである。Claude Code 用に育てた資産をそのまま OpenCode や Cursor Agent に持ち込む、という発想になる。

Pi のように設定ファイルではなく RPC を界面にしている harness は方向性が違う。親プロセスが `pi --mode rpc` を spawn し、stdin/stdout に JSON Lines でコマンドとイベントを流す。モデル呼び出し・ツール実行・コンテキスト管理・認証は Pi 側に残るので、オーケストレータは Node ランタイムの同梱や依存の衝突を避けたまま外から駆動できる。複数 harness を束ねる層を自分で書くなら、この形が扱いやすい。

## 軸3: 権限・安全・コンテキスト

権限モデルは harness ごとに設計思想が違う。

- **Claude Code** — hooks でツール呼び出しに介入する
- **OpenCode** — `opencode.json` の permission rules。読み書きは自動承認、シェルは確認、といった粒度で書ける。[公式ドキュメント](https://opencode.ai/docs/config/)の立場は「Deny より Ask の方が強力」で、エージェントを無力化せずに制御を残すことを狙う
- **OpenClaw** — security level による段階制御。サンドボックス実行を持つ
- **Cursor Agent** — エディタプロセス内で完結する

コンテキスト管理は別軸で、ここは harness の設計が最も分かれる。エージェントの入力トークンは**ツール定義**と**会話履歴**の 2 方向から膨らむ。

ツール定義側を削る手法として、ツールをコンテキストに並べる代わりにコード実行へ逃がす Code Mode / Programmatic Tool Calling がある。API の規模に依らずツール定義のコストを固定できるのが利点で、Claude Code が採る動的ツール検索（関連しそうなツールだけを出す）とはコスト構造が違う。前者は固定費を下げ、後者はヒットしたツール定義の分だけ結局トークンを食う。

ただし Code Mode は生成したコードと実行結果を履歴に積むので、ツール定義から削った分の一部が履歴側へ移る。**両側を同時に測ったデータは、少なくとも私が読んだ範囲では見つかっていない。**

## harness はベンチマークの測定条件でもある

ここからが本題である。

harness を「自分が使うツール」としてだけ見ていると見落とすが、**モデルのベンチマークスコアは、どの harness で測ったかに依存する。** そしてそれは、モデル提供元自身が脚注で認めている。

Alibaba の [Qwen3.8 発表記事](https://qwen.ai/blog?id=qwen3.8)のベンチマーク表の脚注を読むと、こう書かれている。

> Terminal Bench 2.1: Evaluated with Claude Code (avg@10), using a 5-hour timeout and max_tokens=131,072. For all other models, we report the best published score across harnesses

自社モデルは Claude Code harness で測り、他社モデルは**「harness をまたいだ公表スコアのうち最良のもの」**を引用している。つまりこの表の 1 行は、同一条件の比較ではない。

同じ脚注群にはこうもある。

> DeepSWE 1.1: Evaluated with the Claude Code and mini-SWE-agent harnesses ... We report the highest score among both harnesses; notably, Qwen3.8-Max performs best on Claude Code.

**同じモデル・同じベンチマークでも、harness を変えるとスコアが変わる**と明言している。SWE-bench Pro、NL2Repo-Bench、FrontierSWE もいずれも「Evaluated with the Claude Code harness」と注記されている。

興味深いのは、同じ記事が別の箇所では harness 間の差が小さいと主張している点である。

> Fig 2. Qwen3.8-Max achieves comparable performance across many harnesses, including QwenWork, Claude Code, Codex, OpenClaw, and Hermes.

「harness をまたいで comparable」と「notably, performs best on Claude Code」は、真っ向から矛盾はしないが緊張関係にある。comparable の幅がスコア表の順位差より大きければ、表の順位は harness の選択で入れ替わりうる。

この記事は harness を RL 環境のスケーリング軸としても扱っており、Task（単一タスク → 複数タスク → 複数日）、Workspace（複数ファイル → 階層フォルダ → 異種混在フォルダ）、Harness（種類・バージョン・skills）の 3 軸で環境を増やしたと書いている。**harness は測定条件であると同時に学習条件でもある**わけで、そうであればモデルは自分が訓練された harness で最もよく動く、という帰結は自然に思える。

ここから 2 つ言える。

1. **モデル比較記事のスコアを読むときは、harness が揃っているかを脚注で確認する。** 揃っていない表は珍しくない
2. **自分の環境でモデルを乗り換えるとき、ベンチのスコア差はそのまま再現しない。** 自分が使っている harness が、そのモデルの評価に使われたものと違うなら特に

## どう選ぶか

軸を踏まえた実務上の目安。

- **基準を作る** — まず Claude Code か Codex CLI で動作確認する。公式 CLI が最も検証されており、比較の原点になる。OSS での再現性を見るなら OpenCode
- **モデルを切り替えたい / 自前で束ねたい** — OpenCode、Pi、OpenClaw。可搬性で有利。特に外部オーケストレータから駆動するなら Pi の RPC 界面
- **課金・組織統制を挟みたい** — ori harness のようなラッパーで、CLI のコマンド体系を変えずに認証と請求を集約する
- **IDE 内で完結させたい** — Cursor Agent

ただし、この手の機能比較が決め手にならないことも多い。[pi-coding-agent を使わずに OpenCode Go × Hermes Agent を選んだ理由](https://zenn.dev/jodycraft/articles/c35818f2f6c28a)という記事では、選定理由が harness の機能差ではなく「Hermes Agent は Discord に標準対応していて、既にそれが動いていたから」だと明記されている。**周辺環境が整っているかが、しばしば機能差より重い。**

そしてもう一つ。harness はモデルが強くなるほど削れる前提の塊である。Anthropic の [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents) は harness そのものを論じた記事ではないが、その設計原則はここにも効く。

> Consistently, the most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns.

harness の選定は固定的な優劣で決まらない。**「いまのモデルで何を補う必要があるか」**で決まり、モデルが変われば答えも変わる。

## まだ分かっていないこと

正直に書いておく。

- **各 harness の hooks / permission の挙動差を、同一タスクで実測した比較データを見つけられていない。** この記事の軸3 は設計思想の比較であって、性能や安全性の実測ではない
- **コンテキスト圧迫について、ツール定義側と会話履歴側を同時に測ったデータがない。** Code Mode がツール定義から削った分がどれだけ履歴側へ移るのかは、公開データでは追えていない
- **Qwen3.8 の inhouse ベンチ（RecreationBench、E-Commerce Bench 等）は外部検証がまだない。** 上で引いた脚注は harness の話としては一次資料だが、スコアそのものの妥当性は別問題である

harness 差がベンチマークのスコアに与える影響を分離した研究は、少なくとも私は見つけられていない。ここは誰かに測ってほしいところである。

## 参考

- [Qwen3.8-Max — Qwen](https://qwen.ai/blog?id=qwen3.8) — ベンチマーク脚注と Task/Workspace/Harness の 3 軸スケーリング
- [Building Effective AI Agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents) — ワークフローとエージェントの区別、3 つの設計原則、ACI
- [OpenCode Docs — Config](https://opencode.ai/docs/config/) / [Rules](https://opencode.ai/docs/rules/) — permission rules と `AGENTS.md` の扱い
- [pi-coding-agent を使わずに OpenCode Go × Hermes Agent を選んだ理由](https://zenn.dev/jodycraft/articles/c35818f2f6c28a) — 実運用での選定理由とコスト実測
