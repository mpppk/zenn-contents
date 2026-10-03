# Sierra Agent Studio 機能調査まとめ

調査日: 2026-10-03
画像は `images/` 以下に保存した公式公開UIを直接添付する。管理コンソール自体はログイン必須のため、公式ブログ・製品ページの公開画像をキャプチャとして利用する。

## 1. 全体像

Agent Studio は Agent OS のノーコード基盤である。CX、オペレーション、エンジニアが build / test / deploy / optimize を回す。Agent SDK と同じ building blocks をノーコードで扱える点が 2.0 の要点である。

公式: https://sierra.ai/product/agent-studio / https://sierra.ai/blog/agent-studio-2-0

## 2. Journeys

自然言語で定義するステップワークフローである。記事参照、外部API呼び出し、顧客からの情報収集を組み合わせる。チャネルごとの出し分けに対応する。認証リンクを chat で送り確認コードを電話で送る使い分けが例として挙げられている。

保険の保険金請求、医療のトリアージ、ハードウェアのトラブルシュートなどに使われている。スポーツ小売は FAQ、注文追跡、返品、サイズ推奨、リード振り分けの 5 journeys で解決率 75%である。エンタメは予約、ポリシー、店舗情報で chat 86%、voice 79%の解決率である。

2.0 の Journeys は Agent SDK と同じ intelligence で動作する composable building blocks である。逐次ワークフローや SOP の箇条書きを超えて、目標理解、ポリシー遵守、ツール使用、推論を組み合わせる。

![Journeys Transaction Dispute](https://i.gyazo.com/567bde56d2f12bfc0fc66c159184e90b.png)
*Transaction Dispute の Description / Criteria / Guidance と Chat での Agent reasoning 表示。Publish ボタン付き。from https://sierra.ai/blog/meet-agent-studio*

![Journeys 2.0](https://i.gyazo.com/3d7a80a468a36b537f17eb6bf8e1b8b1.png)
*composable building blocks の図。from https://sierra.ai/blog/agent-studio-2-0*

- Agent instructions: ゼロからの定義と既存 operating procedures からの AI 生成に対応する
- Tools and dynamic data: 外部システムや knowledge source の参照により応答を最新に保つ

## 3. Agent Traces

1メッセージごとの意思決定経路を時間付きで表示する。live、手動テスト、simulation の全てで生成される。instructions、tool calls、knowledge lookups、network requests、language guidance を含む。

voice では latency 自体が UX のため、tool 呼び出しと API 呼び出しの所要時間差からボトルネックを特定する用途が強調されている。

デバッグでは、なぜそのツールを選んだか、他に選択肢はあったか、矛盾した指示がなかったか、orchestration logic は正しいかを掘り下げられる。Sierra の building blocks と自前の API 呼び出しの両方を取り込む。ノーコード向け簡易表示と SDK 向け詳細表示を使い分ける。

![Agent Traces timeline](https://i.gyazo.com/ef905090ce857e384a237ae35cb3bd6f.png)
*Transcription 185ms、Supervisor、Tool Call、GET Request、Synthesis 8.83s などの内訳と Decision / Response / Tools available の表示。from https://sierra.ai/blog/agent-traces*

## 4. Knowledge

- Knowledge management: Help Center、FAQ、policies を表示・管理・編集する。既存 sources と JSON 経由の custom articles を接続する。Agent Studio 内で新規 article を直接作成できる
- RAG 最適化: Accuracy (関連情報の優先)、Latency (fine-tuned models、parallelism、smart caching)、Scale (数十万 articles)、Experience (フォローアップ質問や代替案提示)
- Knowledge gaps: 回答が欠けたテーマを自動特定する
- Expert Answers: 対応履歴から articles を自動 draft する

アパレルは国際配送遅延で問い合わせ 65% 増に対し knowledge を即時更新し、再学習なしでサービスレベルを維持した。

## 5. Integrations

- Out-of-the-box 40+: knowledge bases、systems of record、contact centers に即時接続する
- Custom integrations: opinionated framework で Agent Studio 内で設定する
- Agent actions: journeys 内で動作させ、代理 action から care team への routing まで担う
- Integration Library (2.0): credentials と endpoints を追加して publish し、Agent Studio と Agent SDK の両方に tools を露出する。数週間かかった custom 開発が数分になると説明されている。複雑な用途は Agent SDK で拡張できる
- MCP、REST、GraphQL、custom に対応し、ChatGPT Apps へ MCP で 1-click 公開できる

## 6. Simulations

agent、mock user、judge の3者構成である。user は言語、技術習熟度、トーンを変え、ログイン有無や email 可用性などの context を付与する。judge は goal 達成可否、SOP 遵守、brand guideline 遵守、正確性、有用性、分かりやすさを採点する。SOPs、knowledge bases、coaching transcripts、flows から test cases を自動生成し、更新のたびに再実行できる。

- AI-powered evaluations: outcomes に対する評価
- Regression testing: release 前の test suite
- Voice Sims: transcription、noise、話者差への耐性確認

![Simulations Card replacement](https://i.gyazo.com/32b5d0743056f033360a5678bde11753.png)
*Card replacement の User instructions、Expected agent behavior、会話ログの一覧。from https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent*

CX チームは Journeys と並べて扱い、journey 変更の publish 前に pass を求める。developers は GitHub Actions や CLI で CI/CD に接続し、unit tests と同様に releases を gate する。1日あたり35,000件超の実行があり、解決率 90%、CSAT 4.5/5.0 超の事例がある。

## 7. Workspaces (2.0)

自分の Workspace で build and test し、merge して snapshot を作り、QA、staging、production に promote する。journeys、simulations、tools、configuration、knowledge、code を隔離し、conflicts を Agent Studio 内で解決する。監査と rollback を備え、CLI、GitHub Actions、PR紐付け、自動・scheduled promotions に対応する。ノーコードは Agent Studio 内で schedule する。

200 名超で協調する fintech の事例がある。検証は既存 Simulations の実行、新規 journeys の自動生成、Dev Chat での手動対話で行う。Workspace は collaborative draft、feature branch の扱いである。

![Workspaces workflow](https://i.gyazo.com/e83f885d2514cc4fc682f4907eda2602.png)
*individual workspaces -> snapshots -> releases の図。from https://sierra.ai/blog/workspaces*

![Release Snapshot](https://i.gyazo.com/940d109fce33adae3460f0da8f68d6c7.png)
*Release Snapshot dialog with schedule。from https://sierra.ai/blog/workspaces*

## 8. Ghostwriter

エージェントを構築するエージェントである。Agent Studio の journeys、actions、policies、personas は Ghostwriter 生成分も含め可視・編集可能である。Simulations と Experiments で本番前に影響を検証する。

- Build: 振る舞いを記述するだけで workflows、integrations、guardrails、tone を更新する。SOPs、transcripts、SME 音声インタビューから journeys を生成する。live 前に review、approve する
- Test: build、update ごとに自動実行する。自明でない edge を網羅し、失敗診断と次の変更提案を行う
- Improve: conversations、metrics、releases、experiments を goals、guardrails で選別する。Slack、Teams に evidence と next step を提示し、release 後も追跡する。customer context と team feedback から未充足 needs を発掘する

![Ghostwriter build](https://i.gyazo.com/85d17a9e165749f5cdfd30f89710c3be.png)
*Updating agent / Researching / Creating tests / Running tests の表示。from https://sierra.ai/product/ghostwriter*

2026-09-28 の teammate 化では Slack、Teams の channel に追加し、@-mention で resolution rate 動向や sales funnel chart や experiment 起案に応答しつつ能動提案を持ち込む。transfer 要因の特定と 3 changes、影響見積もり、優先 2 件の推奨、wall of love (delight 5 件)、payment failure の backend、integration 分離提案、promo gap の experiment 提案などを行う。不要時は沈黙し、週次で振り返る personality が与えられている。

![Ghostwriter transfer analysis](https://i.gyazo.com/357d9d95ca0b10b4e2c5781dba543682.jpg)
*3,720 calls の review から 3 changes を特定し、影響見積もりと experiment 起案を提示した図。from https://sierra.ai/blog/ghostwriter-ai-tool-to-teammate*

![Ghostwriter A/B test chart](https://i.gyazo.com/270b3a4d6ba0a5d27a1b01962262c5de.jpg)
*experiments の結果が statistically significant かを伝える図。from https://sierra.ai/blog/ghostwriter-ai-tool-to-teammate*

![Ghostwriter payment investigation](https://i.gyazo.com/6adf18d916fa7d36fe2ee356290d47a4.jpg)
*payment failure の原因切り分け図。from https://sierra.ai/blog/ghostwriter-ai-tool-to-teammate*

## 9. Explorer / Insights

Insights は Reporting、Experimentation、Observability で構成される。Reporting は CSAT、case resolution などの key metrics、data point から Explorer で調査、automated tagging である。Experimentation は Value discovery (解約予兆を LTV 機会に転換)、Conversation design (hand-off rules、determinism の variations)、Intelligent decisioning (agent memory、customer profile に基づく personalize) である。

Explorer は weekly briefing を自動配信し、想定外パターンを浮上させる。自然言語で数千会話を横断分析し、summary、categorized themes、根拠会話 links を返す。journeys、channels、time windows で深掘りできる。recommendations を Ghostwriter へ one-click handoff して自動実装し、前後比較で効果測定する。

![Explorer CSAT](https://i.gyazo.com/e5b8d1328c3d3ae28930e333e6e5271e.png)
*CSAT drop への query と suggested questions の UI。from https://sierra.ai/product/explorer*

![Explorer Recommendations](https://i.gyazo.com/a1f333bd13a1c7c50a564e35adfce429.png)
*Recommendations card と Compare findings / Show all conversations。from https://sierra.ai/product/explorer*

Monitoring、Auditing、Alerting は OpenTelemetry、EventBridge、Pub/Sub、export API で外部に送れる。Monitors は hallucination、policy violation、sentiment 急落、abuse を検知して PagerDuty 等に通知する。Traces で個別会話、Explorer で pattern、root cause を調査する。

## 10. Branding / Channels

- Customized agents: 名前、声、welcome message、logo、colors。ThirdLove (Barbra) や SiriusXM (Harmony) が例である
- Dynamic updates: promotion、service update などを real time 反映する
- Multimodality: 画像・動画対応で製品紹介やトラブルシュートを補強する
- Voice Personas: What it does と How it expresses itself を分離し、1 agent に複数 personas を付与する。60 以上の locales の model constellation から最適モデルを選択し、本番で声を sealed する。数千件の Voice Sims と A/B test が可能であり、適切な persona で解決率が約 50% 向上した事例がある
- Channels: voice、messaging、email、Live Assist、ChatGPT を単一 agent で担う。Web chat、mobile app、WhatsApp、Apple Business Chat、SMS、Email、Voice、Live Assist、ChatGPT App に対応し、59 言語に対応する。Build once, deploy everywhere が方針である
- Voice: 低遅延、割り込み、背景雑音、訛り対応、real-time 感情検知で tone、pace を適応し、IVR を置換する
- Messaging: website、mobile などを跨ぐ chat、proactive outreach、画像・動画の multimodal messages である
- Email: inbox を自律応答し、CRM、real-time account context で personalize し、chat 速度で branded 応答する
- Live Assist: 人間への接地時に grounded な ready-to-send 応答を real-time 提案し、agent と同じ tools で即時 action し、全 interaction を feedback loop に入れる
- ChatGPT: 単一 Sierra agent が first-party と ChatGPT app を兼ねる。expose する journeys、data、capabilities を channel ごとに選択し、maps、forms、charts の interactive experiences を作り、MCP native で 1-click、CI/CD 公開する

![Harmony](https://i.gyazo.com/71e1da12ae863e34306edb0c90d0e6f6.png)
*SiriusXM Harmony の例。from https://sierra.ai/blog/meet-agent-studio*

![Email](https://i.gyazo.com/2955485569c8fa3a3a2161a141d1d550.png)
*予約変更3択メールの例。from https://sierra.ai/product/channels*

![Live Assist](https://i.gyazo.com/f4faa9217774958ddbe875d293386bda.png)
*reason for calling / sentiment / benefit を表示する care UI。from https://sierra.ai/product/channels*

![ChatGPT](https://i.gyazo.com/ddd003d5fdc7d1eef7a7739986351f45.png)
*ChatGPT app の listings 例。from https://sierra.ai/product/channels*

## 11. 透明性、ポータビリティ

Your agent, laid bare の要点である。

- See exactly how it works: journeys、actions、policies、personas は Ghostwriter 生成分も含め Agent Studio で可視・編集可能である。Simulations、Experiments で事前検証し、本番後は Traces、Explorer、Monitors、Pulse と外部 observability 連携で追跡する
- Keep the value: logic (journeys、policies、prompts) は portable な構造化 format で export 可能、data、history は export APIs で warehouse、BI に送れる、code は access 可能な Git repository に存在し、roles、permissions、environments、versioned releases で統制する
- Build on what you have: voice、chat、email、APIs を跨ぎ、MCP、REST、GraphQL、custom で既存 systems と接続し、外部の tools を呼び出したり外部 agents から Sierra の capabilities を API 経由で呼べる。1 journey から開始できる

## 12. Cosense 突合せ

- niki-ai、niki-auth、niki-tech で "Sierra Agent Studio" は 0 件である
- niki-cs で "Sierra" は 5 件ヒットである。`Sierra` ページに Agent Studio、Ghostwriter、Horizon、Insights、Observability、Omnichannel の整理がある
- `外部システム連携とアクション実行の比較` に Sierra 行がある。プリビルド40、Integration Framework、REST、MCP、Framework でノーコード、Tools and dynamic data として Journeys に組込と整理されている。実行エンジンは Journeys の Agent instructions で推論し、SoR への deterministic な書き込み、Supervisor models、Agent Traces である

## 13. 制約、未確認事項

- Agent Studio コンソールの実画面 (ログイン後) は未確認である。契約とデモ依頼が必要である
- Knowledge 管理 UI、Integration Library の設定画面、Dev Chat、Voice Sims の詳細操作は公式図のみで手順未確認である
- Trust and reliability (SOC2、ISO、HIPAA、PCI DSS 分離など) と Guardrails、Supervisor models の対応関係は別途整理が必要である

## 参考

- https://sierra.ai/product/agent-studio
- https://sierra.ai/blog/meet-agent-studio
- https://sierra.ai/blog/agent-studio-2-0
- https://sierra.ai/blog/workspaces
- https://sierra.ai/blog/simulations-the-secret-behind-every-great-agent
- https://sierra.ai/blog/agent-traces
- https://sierra.ai/product/ghostwriter
- https://sierra.ai/blog/ghostwriter-ai-tool-to-teammate
- https://sierra.ai/blog/your-agent-laid-bare-and-why-it-matters
- https://sierra.ai/product/insights
- https://sierra.ai/product/explorer
- https://sierra.ai/product/channels
- https://sierra.ai/blog/introducing-voice-personas
- Cosense niki-cs Sierra / 外部システム連携とアクション実行の比較
