# Fin Flywheel (Train → Test → Deploy → Analyze) 機能調査まとめ

調査日: 2026-10-03
対象: Fin (旧 Intercom) の Fin AI Agent を継続改善する 4 ステップのループ (Fin Flywheel)
管理コンソール自体はログイン必須のため、公式サイトとヘルプセンターの公開画像をキャプチャとして利用する。

## 0. 全体像

Fin Flywheel は Train、Test、Deploy、Analyze を回す継続改善ループである。使うほど良くなる設計であり、4 段階が次の段階に送られる。コンソール上の導線は Fin AI Agent > Train、Test、Deploy、Analyze である。

- Train: Procedures、knowledge、policies で複雑な問い合わせに対応できるよう学習させる
- Test: 本番前に開始から終了までの会話全体をシミュレートし、挙動を確認する
- Deploy: voice、email、chat、social の全チャネルで live にする
- Analyze: AI 搭載の Insights で性能を分析・改善する

## 1. Train (学習): Fin に知識と振る舞いと作業を教える

Train セクションでは Content (Fin が知る内容)、Guidance (Fin の振る舞い)、Tasks、Procedures (Fin が行う作業) を扱う。Content と Guidance のページには Preview パネルがあり、会話を試しながら Customer view と Event log を確認できる。

### 1-1. Procedures: 複雑な処理を自然言語と決定論的制御で定義する

Procedures は Tasks の後継であり、返金、サブスク更新、注文追跡などの多段階処理を担う。自然言語の手順書のように記述し、if/else 分岐やコード (日付計算、適格性検証、レコード更新) で決定論的制御を付与する。会話は非線形のため毎ターン推論し、適切なステップへ skip、切替する。Stripe、Shopify、Linear などの外部システムとは Data connectors、MCP で連携する。共通手順は sub-procedures として再利用できる。逐次実行であり、並列処理には対応しない。

![Procedure editor](https://i.gyazo.com/108a2f7c68f33ab1249a81429c952a85.png)
*Procedure: Withdrawal request の編集画面。When to use this procedure、対象 (Everyone on Web, iOS, Android)、手順内の Use: Verify identity などの Data Connector 選択、右上の Guidance、Test、Save、Set live。from https://fin.ai/train*

操作の流れは Train > Tasks (Procedures) で New task を開き、trigger の title と description (使う場面と使わない場面を 3-5 文で記述)、Trigger when と Don't trigger when の example questions (10 件程度)、channels と audience rules、Give Fin instructions の手順ブロック (動詞始まり、if + else 形式)、Data connectors (@ で Connect with external system)、attributes と temporary attributes、task 内 guidance、Wait for webhook、会話タグ付け、Escalate to the team を設定する。AI-generated task instructions では説明文から構造化プロンプトを自動生成できる。live task の編集は Save as draft で draft 版を作り、live 版に影響させず Preview と Simulations で検証してから Set live する。

![Tasks list](https://i.gyazo.com/8c52b01e9994e55393a2ab9bd8b5af81.png)
*Train > Tasks の一覧画面。左ナビの Train 配下に Content、Guidance、Escalation、Tasks、Suggestions が並び、右上に New task がある。from Fin Tasks ヘルプ*

### 1-2. Content: 回答の根拠を与える

Help Center 記事、内部ドキュメント、PDF、URL、Zendesk 同期などの knowledge source を Content library で一元管理する。Fin は複数 source から関連情報を組み立てて回答する。Audiences では plan、location、brand などで content を出し分ける。

![Content](https://i.gyazo.com/22e2de302d97c23455aa52793a70c254.png)
*Add your existing content の図。from https://fin.ai/train*

### 1-3. Guidance: 話し方と policies を指示する

tone of voice (professional、friendly、humorous など) と回答の長さを選び、自然言語の instructions で brand の声、policies、問い合わせの捌き方を教える。escalation の条件も deterministic rules と自然言語 guidance の両方で定義できる。personality 設定は Guidance ページに移管されている。

![Guidance](https://i.gyazo.com/1a178023c39552f67eaa2b80452c32a0.png)
*Define Fin's behavior の図。from https://fin.ai/train*

### 1-4. Suggestions と Data connectors

Suggestions は解決できなかった会話を元に content の改善案 (更新、新規作成、重複指摘) を AI が提示する。Intercom、Zendesk、Salesforce 横断で動作し、却下からも学習する。Data connectors は外部システムへの接続であり、回答の personalization と Tasks、Procedures 内の action 実行に使う。IDV 有効時は name、email、city、country などの基本属性を自動取得する。Vision は画像 (スクリーンショット、請求書、エラー画面) の読解であり、45 言語の real-time translation に対応する。

![Data Connectors](https://i.gyazo.com/b2e815b2f1ee3936c3df66c5ed4ffdb0.png)
*Linear、Stripe、Shopify などへの MCP、API Data Connectors の図。from https://fin.ai/train*

## 2. Test (テスト): 本番前に会話全体を検証する

Test では Fin AI Agent > Test で customer questions を投げ、回答の sources と settings を確認し、改善 recommendations を受ける。batch test は inbox の会話取り込み、手動追加で accuracy と performance を評価する。Answer rating は回答ごとに評価し、report に集約する。Fin preview は guidance、deployment settings、intro message の変更を Customer view で即時確認し、audiences 別の見え方も試せる。Answer inspection は回答生成に使われた sources と settings (tone of voice、Guidance) を全表示する。

### 2-1. Simulations: AI customer と AI judge で合否判定する

Simulations は実 scenario を開始から終了まで全会話で再現する。AI が simulated customer 役となり、指定 context で Fin と会話し、別の AI が success criteria に基づき judge して pass、fail を付ける。失敗時は transcript で原因を追える。保存した Simulations は Procedure 更新のたびに再実行し、regression を検知する。

![Simulation creation](https://i.gyazo.com/67608068cba01b090a723490bb324123.png)
*Shipping damage の simulation 作成画面。左に Instructions ブロック (Get orders、条件分岐、全額返金など)、右に Test as (Preview user、All brands)、Customer's starting message、follow-up messages、Customer data available to Fin、Save、Run がある。from Fin Tasks ヘルプ*

操作の流れは Train > Tasks で task を edit し、Instructions ブロックの Test から New simulation を開き、Test title、Test as (workspace の contacts)、user の opening message、User context (状況の補足)、Available data (data connectors、attributes の値)、Success criteria (Fin が理由を説明する、next step を提示する、data connector が発火するなど) を入力し、Save または Run する。実行結果は右の Tests パネルに Passed、Failed、Not yet run で並び、See conversations で simulated customer と Fin の往復を確認する。概要ページからの Simulations は live 版、editor 内からは draft 版を対象とする。

### 2-2. Eval-driven delivery: Evals、Releases、Monitors

AI 研究の eval 手法を CS 運用に移植した考え方であり、Test ステップの中核である。Evals は実会話、低 performance Topic、失敗会話から数千 scenario を AI が自動生成し、1 つの LLM が顧客役、別が judge 役でハンドオフの timing、content、data の正しさを評価する。重要 scenario は保存して変更の度、定期実行する。Releases は本番から隔離された専用 space で変更を共同編集し、simulation から本番 A/B test で影響検証し、段階 rollout、即時 rollback する。Monitors は本番の全会話を品質基準で継続評価し、閾値超過で alert し、逸脱会話をワンクリックで次回 Eval 化する。Operator は Evals 構築・実行、Release draft、Monitors 設定を自律実行し、人間は review、approve のみ行う。

## 3. Deploy (展開・導入): チャネルと対象を決めて live にする

Deploy セクションでは live chat、email、phone の各チャネルに Fin を出す。手段は Simple deployment と Workflows であり、Fin AI Agent > Deploy > Chat、Email、Voice で設定する。一度更新すれば全チャネルに即時反映する (Configure once, apply everywhere)。

![Deploy channels](https://i.gyazo.com/67591da38cae8f8c409cbacd895cbed3.png)
*Facebook、電話、WhatsApp、Intercom、Gmail、Instagram、Slack のチャネルアイコンと、chat、Fin チャット、email (Scheduling food order) の 3 画面。from Fin AI Agent explained*

- Fin over chat: Messenger、WhatsApp、SMS、social で挨拶、即時回答、escalation する
- Fin over email: inbound email を非同期解釈し、content から回答し、phishing、spam を除外し、履歴全体の context を保持する
- Fin over phone (Fin Voice): 複数言語で自然応答し、24/7 対応し、必要時に人間に接続する。Twilio、Genesys などの既存 telephony に転送、SIP 統合する
- Fin over Slack、Discord: スレッド内で応答し、trigger、workflow で協働制御する
- Workflows for Fin: automaton に Fin を追加し、triage と複雑 query 対応を行う。追加 setup は不要である
- Outbound Fin: FullStory、Pendo の behavioral signal (rage click など) を起点に proactive 会話を開始する。Proactive Procedures では Web 操作や外部 signal を trigger に Fin 側から開始する
- Human handoff: triage と handoff の条件を設定する。自傷、 jailbreak、高 risk 医療・法律・金融助言などでは自動 handoff する
- Audience targeting: audience、region、channel で出し分ける
- Usage limits: resolution 上限で通知、停止する
- Fin over API: 自社 messenger や help center 検索に組み込む

## 4. Analyze (分析): 性能を見て次の Train につなげる

Analyze セクションは live 後の real-time Insights であり、継続改善の起点である。学びを元に train、test、deploy を回して対応 volume を増やす。

![Performance dashboard](https://i.gyazo.com/1ca27bf7edbd582ccb010d78728c1795.png)
*Performance 画面。Automation rate 55%、CX Score 63%、Involvement rate 73%、Resolution rate 76%、Performance funnel。from https://fin.ai/analyze*

### 4-1. Insights: 全会話を常時分析する

- Performance dashboard: resolution rate、involvement rate、CX Score を一画面に集約する。Fin の対応範囲、解決、顧客体験を追い、 Fin AI Agent > Analyze > Performance で確認する (customize 不可、Reports から custom report は別途作成する)
- Optimize dashboard: 改善領域を AI が提示し、review、approve で数秒で反映する
- CX Score: survey なしで全会話の sentiment を AI 評価する。CSAT の 5 倍の coverage である
- Topics Explorer: 会話を topics、subtopics に自動分類し、volume の要因を特定する。Topics、Subtopics の作成、merge、削除、改名に対応する
- Trends: 週次の volume 急増、性能低下、新規質問を自動 report する

![CX Score](https://i.gyazo.com/b5438b629ea3d3d3176712e67f9a84b5.png)
*CX Score dashboard。from https://fin.ai/analyze*

![Topics Explorer](https://i.gyazo.com/cac3c6a2c76b363896abb69ba5af7b82.png)
*Topics Explorer の volume 表示。from https://fin.ai/analyze*

### 4-2. Monitors: 品質を継続評価する

Monitors は Fin と人間の全会話を standards 照合で評価する。filters と自然言語で監視対象を定義し、Custom Scorecards で自社基準の良い対応を定義して AI が全会話を採点し、Alerts で逸脱を real-time 通知し、Review Queues で対応を一元管理する。Monitor reports は Analyze > Custom Reports > Chart の Monitors 指標で作成する。

![Monitors](https://i.gyazo.com/2aeedc80fcecdd874205bfb899c4e59d.png)
*Monitors overview と edit monitor panel。from https://fin.ai/analyze*

### 4-3. Recommendations: 未解決から優先 fix を提示する

Recommendations は Fin が解決できなかった会話を全件解析し、content、data、action の gaps を影響度順で提示し、one-click で適用する。Anthropic は週次 review で総解決率 15% 向上した。Fin AI Agent > Analyze > Conversations から inbox の Fin 会話に飛べる。holistic reporting は AI と人間を unified view で示し、custom reporting は chart 作成と drill-in に対応する。

![Recommendations](https://i.gyazo.com/c9162e12b23acf652d8d9c9a36464a92.png)
*優先度付き改善提案の一覧。from https://fin.ai/analyze*

## 参考

- https://www.intercom.com/ai (Fin Flywheel 概要)
- https://fin.ai/service
- https://fin.ai/train
- https://fin.ai/testing (Eval-driven delivery: Evals、Releases、Monitors、Operator)
- https://fin.ai/analyze (Insights、Monitors、Recommendations)
- https://fin.ai/procedures
- https://www.intercom.com/help/en/articles/7120684-fin-ai-agent-explained (Train、Test、Deploy、Analyze の capabilities 一覧)
- https://fin.ai/help/en/articles/13976187-fin-procedures-explained
- https://fin.ai/help/en/articles/13975775-how-to-set-up-fin-tasks
- https://fin.ai/help/en/articles/13975849-fin-tasks-best-practices-and-examples
- https://fin.ai/blueprint/service/launching-ai-agents/evaluate
- Cosense niki-cs Fin: https://scrapbox.io/niki-cs/Fin
- Cosense niki-cs Eval-driven delivery: 概念ページ (Evals、Releases、Monitors の3点束ね)
- Cosense niki-cs 外部システム連携とアクション実行の比較: https://scrapbox.io/niki-cs/外部システム連携とアクション実行の比較 (Fin 行: Data Connectors 37 + Helpdesk 5 + Channels 9、計51、MCP、API、OAuth2.0、Custom MCP)

## 制約、未確認事項

- Fin コンソールの実画面 (ログイン後) は未確認である。Test の batch test 画面、Deploy の Simple deployment 設定画面、Analyze の Optimize dashboard の操作手順はヘルプの記述のみであり、 clicks の厳密な順序は未検証である
- Fin Tasks は managed availability であり、Procedures への移行が進んでいる。Tasks と Procedures の画面差分は Transitioning from Fin Tasks to Procedures の確認が必要である
- niki-ai、niki-auth、niki-tech で "Fin Flywheel" は 0 件であり、niki-cs の Fin ページが主参照である
