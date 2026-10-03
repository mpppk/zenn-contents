# τ-knowledge まとめ

出典: https://taubench.com/blog/tau-knowledge.html
著者: Ben Shi, Ola Zytek, Pedram Razavi (Sierra, 2026年2月)
関連: 論文 https://arxiv.org/abs/2603.04370 / コード https://github.com/sierra-research/tau2-bench

## TL;DR

- τ-knowledge は knowledge-intensive なカスタマーサポートを評価するベンチマーク。τ-Bench の拡張。
- 新ドメイン τ-Banking (fintech風カスタマーサポート) + 現実的な KB (698ドキュメント、21商品カテゴリ、約195Kトークン) で構成。
- タスク成功には multi-step reasoning、ポリシー適用、tool use の連携が必要。
- 最高スコアは GPT-5.2 high reasoning でも約26% pass^1。現行フロンティアモデルは messy な実ドキュメントの検索・解釈・行動に失敗する。

## 背景・モチベーション

- 現代のエージェントは、整形済みコンテキストではなく、大きく messy で変化し続ける KB 上での動作を期待される。対象は内部ドキュメント、ポリシーマニュアル、ツール説明、商品カタログ、手順書など。
- この種の KB の難しさ:
  - 情報が非構造化で長文ドキュメントに分散
  - ポリシーが手続き的・条件付きで誤適用しやすい
  - time-sensitive (例外・プロモーションで挙動が変わる)
  - 商品名や内部用語が out-of-distribution で embedding 前提を崩す
  - 一部ツールが discoverable (エージェントに直接渡されず KB ドキュメント内でのみ参照される)
- 既存ベンチマークは retrieval (QA) と tool use を分離して評価しがちで、live のユーザー対話中に private KB を参照しつつツールを協調させる設定を捉えていない。

## τ-Knowledge / τ-Banking の概要

- τ-Banking は fintech 由来のカスタマーサポート設定。タスク成功は KB からの情報を正しく解釈・運用できるかに依存。
- factoid 型 RAG との違い: 正しいドキュメントを取ってくれば答えが出るのではなく、KB の証拠とツール出力を長期会話にわたって突き合わせて、検証可能な DB レベルの状態遷移を起こす必要がある。
- retrieval mechanism agnostic。dense / sparse / hybrid / long-context / terminal 経由の filesystem 探索など任意の戦略を許容・評価する。従来の semantic retrieval を超えたパラダイムも評価できる。
- タスクあたり平均 18.6 必須ドキュメント、9.5 必須ツールコール。最大33ツールコールのタスクもある。

### 3つの設計要素

1. Discoverable tools
   - 多くのツールはデフォルトで使えず、KB ドキュメント内に暗黙的に参照されているのみ。
   - 使うにはドキュメントを特定し、unlock してから invoke する。実デプロイでエージェント能力がハードコードではなくドキュメントで定義される状況を模倣。
   - 記事の例: `request_credit_limit_increase_4829` や `file_dispute_8291` を `unlock_discoverable_agent_tool` してから `call_discoverable_agent_tool` する。unlock を飛ばすとエラーになり、成功を hallucinate する失敗がある。
2. Flow-based user simulation
   - 各タスクが条件分岐ルールを持ち、評価上重要な分岐でシミュレートユーザーの振る舞いを規定。エッジケースへの誘導、不適格要求の拒否テスト、会話途中の状態変化への適応などを試す。
3. Objective verification
   - 会話の質ではなく target database state で成否判定。正しい最終システム状態を作ったときのみ成功。

### 記事中の具体例 (Tasks 53 & 27 ベース: Dispute + Credit Limit Increase)

- ユーザーが dispute と credit limit increase を同時依頼。
- KB 検索で「pending disputes があると credit limit request は自動 reject」という依存関係を発見するのが要点。
- 正解順序: 先に credit limit increase を処理 (approve, 新limit $15,000) してから dispute を file (UNDER_REVIEW)。
- 典型失敗:
  - critical policy dependency を見逃し、ユーザー提示順に dispute を先に file → credit limit が DENIED。
  - unlock ステップを飛ばして discoverable tool を直接 call → ERROR 後に成功を hallucinate。
  - 後の会話でユーザーが「dispute が approved された」と主張した際に `get_dispute_status` で検証せず credit を適用。実際は UNDER_REVIEW のまま。

## KB の中身

- 698ドキュメント、21商品カテゴリ、約195Kトークン。
- 対象: 個人・ビジネス checking、tiered savings、rewards credit cards、BNPL など。
- 顧客向け仕様 (APY、金利、手数料、cashback構造) だけでなく内部エージェント手順 (replacement card 発注、口座解約条件、referral ルール、本人確認フローなど) も含む。

## 構築プロセス (LLM生成 + 人手 refine、5ステージ)

使用LLMは GPT-5, GPT-5.2, Claude-4.5-Opus, Gemini-3-Pro の4種を混ぜて文体・構造の多様性を確保。

1. Stage 1: Structured database generation
   - ビジネスカテゴリ (credit cards, savings など) → カテゴリ内 feature (card tier, account protocol など) → 具体値 (annual fee, cashback rate など) を LLM で生成。typed variables の集合としての構造化DB。
2. Stage 2: Structured → unstructured 変換
   - feature ごとに plausible なタイトル (例: "Bronze Rewards Card Overview", "How do I view my monthly cashback?") を生成し、variables をドキュメントに割り当てて自然文の記事化。実 CS の KB らしくする。
3. Stage 3: Task and database creation
   - fintech CS フロー (replacement card 発注、dispute、口座推薦など) を模したタスクと DB を人手+LLM支援で共構築。各タスクは特定ワークフロー中心で、必要な KB 記事・ツールを更新して支援。
4. Stage 4: Human-in-the-loop refinement
   - タスク作成に伴い構造化KBを加除修正し、影響箇所のみ生成パイプラインを再実行。明確さ・現実性のための手編集も実施。
5. Stage 5: Review
   - タスク作成に関与していない2名が独立監査。期待最終状態の正しさ、gold document set の完全性・最小性、それだけで完遂可能かを確認。
- このパイプラインはスケーラブルで feature 間の意図せぬ衝突を抑え、各タスクを KB 変数上の制約集合として表現・検証できるのが利点とされる。

## サンプルタスク3件 (記事より)

1. Recommendation (Multi-Constraint Product Recommendation)
   - Yumi Tanaka が business checking 1件 + business savings 1件の best fit を要求 (比較提示なし)。
   - checking 条件: mobile deposit ≥ $10K/day、overdraft fee ゼロ、minimum balance < $10K、APY ≥ 1%。savings 条件: same-day ACH、minimum balance < $50K、wire fee ≤ $15。
   - フィルタ後に各3件残り、simulation date 11/14/2025 での time-sensitive promotion ポリシー (期間が異なる4件) で tie-break。候補 checking 8 / savings 7、必須ツールコール 6。
   - 期待動作: verification log → `get_all_user_accounts_by_user_id` → `open_bank_account` で business_checking "Sky Blue" と business_savings "Gold Saver Account" を開設。
2. Procedural (Credit Card Retention Protocol)
   - Yuki Nakamura が年会費を理由に Platinum Rewards Card を解約希望。KB の 4ステップ Retention Protocol 遵守が必須。必須ツールコール15、顧客 tenure 3年。
   - (1) 解約適格性確認 (dispute、pending replacement、account age、$75残債の発見・支払い)、(2) 過去1年の retention 試行確認、(3) 理由聴取・log、(4) 理由が annual fee かつ3年以上顧客なので年会費1年 waiver を提示。承諾される前提で正しい expiration date で flag 適用。順序違い・抜けは失敗。
3. Sequencing (Operation Sequencing with Dependencies)
   - Jordan Chen が (1) Bronze savings 解約、(2) business checking 開設、(3) Evergreen checking 解約、(4) personal savings 開設を要求。ユーザー提示順では全失敗。
   - 隠れ依存3件: どれか解約して CLOSED 状態を作ると business checking 申請が block ("no accounts with status CLOSED")、Evergreen 解約後は12日齢口座のみ残り savings 開設の14日 tenure 要件を満たせない。
   - 正解は別KBドキュメントから依存を発見し reorder・説明して13ツールコールを正順で実行: opens first (business_checking "Navy Blue"、savings "Silver Plus")、then closures (Bronze の資金移動→解約、Evergreen の資金移動×2→最後に解約)。

## 実験設定

- 評価モデル: GPT-5.2, Claude-4.5-Opus, Claude-4.5-Sonnet, Gemini-3-Pro, Gemini-3-Flash。
- retrieval 構成:
  - Dense: text-embedding-3-large, Qwen3-Embedding-8B
  - Sparse: BM25
  - Terminal use: KB をファイル export し grep/cat/find 等の shell で探索
  - Golden retriever: ground-truth ドキュメントを直接コンテキストに入れて retrieval を除外
- 指標: pass^k (k回の独立試行すべて成功する確率)、k=4まで評価。

## 主要結果・Key Findings

- τ-knowledge は hard。最高は GPT-5.2 high reasoning で 25.52% pass^1。k を増やすと急落し pass^4 は 13.40%。
- gold documents を直接渡しても最高は Claude-4.5-Opus-High で 39.69% pass^1、pass^4 26.80%。検索だけでなく推論が必要であることを示す。
- 既存 τ-bench ドメインとの比較 (ドメイン別 best pass^1): τ-telecom (Qwen3.5) 97.8、τ-airline (Opus 4.5) 84.0、τ-retail (Qwen3.5) 82.9 に対し τ-banking (GPT-5.2) 25.5。3〜4倍難しい。
- Reliability と efficiency の乖離:
  - Claude-4.5-Sonnet は pass^1 で Opus に劣るが試行間の落ちが小さく pass^4 では逆転 (10.31% vs 9.28%)。
  - Claude 系は GPT 系と同等性能を短い duration で達成。生成トークン 0.7M vs 1.2M、retrieval calls 平均 8.7 (Opus) vs 18.5 (GPT-5.2 high)。
- Freeform search の優位:
  - Terminal-based retrieval が6モデル構成中5構成で最高 pass^1。平均2.4〜3.6ポイント標準 retrieval を上回る。
  - 検索頻度: dense 9.9〜10.1回/タスク、BM25 11.4回、terminal grep 14.5回。terminal は median turn time +6.6秒 (dense比)。
- Solution efficiency を一級指標にすべきという主張。余分なターンは解決時間・認知負荷・信頼低下につながり、特に紛失カードや不正取引など time-sensitive な支援で重要。

## エージェントの失敗モード4種

1. Complex product interdependencies: 商品・ポリシーの深い相互依存で multi-hop 推論が必要。プロモーションボーナスを追って base rate の良い代替を見逃し、表層条件は満たすが suboptimal な組合せを推薦。
2. Implicit subtask ordering: 隠れ依存の順序付け (dispute 先行で credit limit が auto-reject 等)。ユーザー提示順に実行しがち。
3. Overtrusting user assertions: 「dispute は全部 approved」等の主張を system state で検証せず credit 適用などに進む。
4. Search inefficiency and assumptions: 曖昧さを質問・追加検索で解消せず仮定する。例: 「最高 referral bonus の口座は?」で account type 不明なまま credit card と決めつけ、他種別の referral ドキュメントを見ない。

## 記事の締め (Looking Forward)

- スコアの低さこそ要点。最良で約26% pass^1 と改善余地が大きい。
- Web 検索と異なり finite・closed な KB での評価であり、これが実デプロイ (CS、社内ツール、エンタープライズ、規制業種) の現実。数百ドキュメントを捌けないエージェントは open-ended retrieval で信頼できない。
- モデル提供者・開発者に search・reason・act 能力の指標として利用を呼びかけ。ベンチマークはオープン、タスクは検証可能、現状と信頼できるデプロイのギャップは明確。
