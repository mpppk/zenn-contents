# Sierra Context Engineering: Type of Context の用語と役割・UI

調査日: 2026-10-03
対象: https://sierra.ai/jp/blog/context-engineering-the-key-to-great-agents (2026-05-05、Neil Rahilly)
画像は `images/` 以下に保存した公式公開UIを直接添付する。

## 前提: Context engineering とは何か

Context engineering は、各時点でエージェントがどの情報にアクセスし、いつ使うかを決めることである。背景には progressive disclosure があり、各時点では最小限の関連情報だけを渡す。トークンが増えるほど想起精度が落ちるためである。例として、欧州への国際配送の問い合わせでは、送り先が分かるまで国別ルールは noise であり、ドイツ行きと分かってからドイツ向け guidance が必要になる。

Conditions は progressive disclosure を機能させる結合部であり、この情報がどの状況で関連するかに答える。state (tool が特定データを返す、顧客が認証済み、subscription が load 済み) と observation (topic 言及、解約希望、製品質問) に基づく。条件が満たされると情報がエージェントに渡る。会話開始時は最小構成 (basic tools、general policies、brand voice) で始まり、認証後に account-specific tools と policies が利用可能になり、charge への質問で dispute workflow、policies、tools が現れる。各ステップが次に必要なものを unlock する。

![Architecture diagram](https://i.gyazo.com/c0292edb64854890530c5b9ae2cd3952.png)
*rigid な rules-based flowchart (X) と dynamic な goal-driven + policy-based actions (チェック) の対比図。from 上記ブログ*

Sierra はエージェントを condition 付きの composable context blocks の集合として表現する。作り方は Ghostwriter (自然言語指示や SOP、call transcripts、documentation から blocks と conditions を自動生成)、Journeys (ノーコード editor で Ghostwriter 生成分の確認・修正、直接構築)、Agent SDK (コード管理、custom blocks、arbitrary code) である。中身の動作は作り方によらず同じである。

## Type of Context 一覧

ブログの表に 8 ブロックがある。各用語の役割と UI 上の登場箇所を以下に整理する。

### 1. Journey: エージェントが追求方法を知る goal

役割は goal であり、例として dispute a charge、file a claim、book a flight がある。各 journey は trigger と outcome を持つ。

UI 上では Agent Studio の Journeys editor に登場する。Transaction Dispute の画面では Description (The customer mentions a suspicious, unrecognized, or incorrect charge) が発動対象の説明、Criteria (Help the customer resolve a disputed transaction) が trigger 側の定義、Guidance の 1-7 手順が pursuit の中身に対応する。Tools は手順文中に UserAuthentication、LookupTransaction、CheckForFraud のチップとして埋め込まれる。実行時には Chat 横の Agent reasoning の Decisions に Activating Journey: Transaction Dispute と表示される。

![Journeys editor](https://i.gyazo.com/567bde56d2f12bfc0fc66c159184e90b.png)
*Transaction Dispute の Description、Criteria、Guidance と Agent reasoning。from https://sierra.ai/blog/meet-agent-studio*

### 2. Tool: 外部システムとの接点

役割は外部システムとの interaction であり、例として pull an itinerary、check coverage、process a refund がある。

UI 上では 3 箇所に登場する。Journeys editor の手順文中のチップ (Use: Verify identity などと同型)、Agent Traces の Tool Call: SearchKnowledge、GetAccountBalance、GET Request: balance_lookup の行と Tools available 12 (GetAccountBalance、CancelTransaction) の一覧、Integration Library の設定 (library 選択、credentials と endpoints 追加、publish) である。接続後は Agent Studio と Agent SDK の両方に tools として露出する。

![Agent Traces](https://i.gyazo.com/ef905090ce857e384a237ae35cb3bd6f.png)
*Tool Call と GET Request の行、Tools available の一覧。from https://sierra.ai/blog/agent-traces*

### 3. Rule / Policy: 自然言語の guardrails と business logic

役割は guardrails と business logic であり、例として Premium cardholders waive foreign transaction fees がある。

UI 上では 2 層で現れる。定義側は Agent SDK の Goals and guardrails (objectives と guardrails を設定し、policy guidelines と brand tone を守らせる) であり、Agent Studio 2.0 の Journeys は SDK と同じ building blocks を使う。実行側は Agent Traces と Agent reasoning の Supervisors であり、Supervisor: DetectAbuse、Evaluate Conditions、Tool Input Validation、Safety Checks の行として可視化される。Decisions と Responses (Main response) と並ぶ。会話開始時の general policies がここにあたり、認証後に account-specific policies が追加される構成である。

### 4. Workflow: 特定順序が必要な場合の step-by-step guidance

役割は特定順序を要する場面の段階的 guidance であり、例として regulated intake、multi-step verification がある。rigid な flow との違いは、全体の organizing paradigm ではなく、conditions 満了時に渡される context の一部になる点である。

UI 上では Journeys editor の Guidance の番号付き手順が該当の見た目である。Transaction Dispute の 1. Greet から 7. CheckForFraud の手順がその形式であり、認証や照会の順序を固定したい場面で使う。charge への質問で dispute workflow が現れるというブログの例と対応する。

### 5. Knowledge: オンデマンド参照の articles 群

役割は Help center articles、product docs、FAQs、internal policies であり、on demand でアクセスする。

UI 上では Knowledge management (Help Center、FAQ、policies の表示・管理・編集) に登場する。実行時には Traces の Tool Call: SearchKnowledge の行として現れる。RAG は Accuracy、Latency、Scale、Experience の軸で最適化される。Knowledge 管理画面自体の公開キャプチャは未確認である。

### 6. Memory: 顧客の履歴

役割は customer history であり、past conversations、preferences、prior issues を含む。

UI 上では Context Engine 製品ページの Memory カードに登場する。下図では Marcus Webb の Reason for calling: Claim status update、Current sentiment: Negative、Current claim: #CL-40217、Last conversation (water damage の claim、査定は木曜) が表示される。Continuity (チャネルをまたいで前回の続きから対応)、Customer data (基幹システム連携で購入履歴やアカウント情報にアクセス)、Data sovereignty (permissioned、保持、warehouse への export) が機能である。長期記憶の層は Agent Data Platform (ADP) と Horizon が担う。

![Memory UI](https://i.gyazo.com/f95d3ebba9a09dba889cf064b407c7cc.png)
*Memory カード。顧客特定情報と前回会話の要約。from https://sierra.ai/jp/product/context-engine*

Conditions の state 側の具体例として、Signals の History 画面がある。Billing update (Plan price $13.99 → $18.99)、Engagement (Watch time 10.4 hrs vs 30.2 avg)、Signal deduced (Cancel intent Elevated、Inputs: Price change、pause inquiry、engagement trend)、Proactive outreach (Hi Tina、新 episode の通知) が並ぶ。tool が返すデータや behavior が conditions の入力になる流れに対応する。

![Signals history](https://i.gyazo.com/b2ecce7b0555287fd5132e702422e473.png)
*History。Billing update、Engagement、Signal deduced、Proactive outreach。from https://sierra.ai/jp/product/context-engine*

### 7. Glossary: 事業の用語集

役割は business の terminology であり、product names、plan tiers、internal jargon を含む。

UI 上の登場箇所は公開情報では未確認である。knowledge や guidance の中で参照されるものと見られるが、単独の Glossary 設定画面のキャプチャは公式ブログと製品ページにない。未確認として残す。

### 8. Response phrasing: brand voice と tone

役割は brand voice と tone である。会話開始時の最小構成に含まれる。

UI 上では Branding and controls の Customized agents (名前、声、welcome message、logo、colors) に登場する。ThirdLove (Barbra) や SiriusXM (Harmony) が例である。Voice Personas は処理内容と表現の分離であり、1 agent に複数 personas を付与し、60 以上の locales から選択する。

![Harmony](https://i.gyazo.com/71e1da12ae863e34306edb0c90d0e6f6.png)
*SiriusXM Harmony の例。from https://sierra.ai/blog/meet-agent-studio*

## なぜ重要か (ブログの主張)

5 journeys のエージェントは loose な context 管理で動くが、50 のエージェントは各 context が正確な時点で届く必要がある。関連度の高い少ないトークンを送ると hallucination が減り、naturalness と performance が上がり、単純な rebooking で baggage policy の 1000 トークンを処理する費用もなくなる。logic を hardcode するとモデルの能力を事前定義の path に縛るが、context engineering では自由に reason し、新モデルの改善を継承する。

## 参考

- https://sierra.ai/jp/blog/context-engineering-the-key-to-great-agents (EN: https://sierra.ai/blog/context-engineering-the-key-to-great-agents)
- https://sierra.ai/jp/product/context-engine (EN: https://sierra.ai/product/context-engine)
- https://sierra.ai/blog/agent-studio-2-0 (Journeys、Workspaces、Integration Library)
- https://sierra.ai/blog/meet-agent-studio (Journeys 画面)
- https://sierra.ai/blog/agent-traces (Traces 画面、Supervisors)
- https://sierra.ai/blog/agent-os-2-0 (ADP は memory と intelligence の層)
- https://sierra.ai/blog/agent-development-life-cycle (declarative goals and guardrails)
- Cosense niki-ai エージェントのコンテキスト圧迫への対処 (関連の可能性あり、未読)

## 制約、未確認事項

- Agent Studio と Context Engine のコンソール実画面 (ログイン後) は未確認である。Glossary の単独設定画面、Knowledge 管理画面、Conditions の編集 UI は公開キャプチャがなく、SDK 側の定義方法 (Agent SDK docs は契約顧客限定) も未検証である
- Rule / Policy と Journey Guidance の境界 (policies が Journeys editor のどこに書かれるか) は公開情報から断定できない
- niki-ai のエージェントのコンテキスト圧迫への対処ページは未読であり、突合せが残っている
