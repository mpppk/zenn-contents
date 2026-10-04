# memo

## Q&A

### Q. Agent SDKで開発したアプリケーションはどこで動かすのか?

Sierra の Agent OS 上で動かす。Agent SDK は platform as a service であり、開発自体は自社の programming environment と SDLC (version control、CI/CD、audit など) で行うが、runtime は Sierra がホストする。Build once, deploy everywhere の方針で、chat、phone、email、SMS、messaging (および ChatGPT app、contact center) に展開する。Workspaces の snapshots を QA、staging、production の environments に promote する流れである。自前インフラでの self-host は公開情報になく、SDK ドキュメント自体が契約顧客限定である。

参考: https://sierra.ai/product/agent-sdk / https://sierra.ai/blog/serving-customer-experience-and-engineering-teams-all-from-one-platform / https://sierra.ai/blog/workspaces

### Q. Agent SDKを動作させるプラットフォームがAgent OSという名前なのか? 契約者限定のCloudflare WorkersやAWS Lambdaのようなものか?

Agent OS はその名前である。ただし Workers や Lambda のような汎用 compute ではなく、エージェント用の managed platform 全体の名前である。Agent Studio、Agent SDK、Insights (Explorer、Monitors、Experiments)、Channels、Agent Data Platform (memory 層) などを含む。SDK で書いた journeys は Sierra がホストするこの上で動き、channels に展開される。契約者限定・公開 signup なしの点と、Sierra が実行環境を持つ点は Workers、Lambda との共通点である。違いは、汎用コード実行ではなく channels、knowledge、memory、simulations、observability を備えた agent 専用 runtime である点と、compute 従量ではなく outcome-based の料金である点である。Agent OS という名前の無関係な OSS もあるため混同に注意する。

参考: https://sierra.ai/blog/agent-os-2-0 / https://sierra.ai/blog/meet-agent-studio

### Q. Agent SDKはなんの言語で利用できるのか? 汎用コード実行できないとしたら、どんなAPIなのか?

対応言語は公開情報では不明である。公式の表現は declarative programming language と existing programming environment であり、SDK ドキュメント自体が契約顧客限定のため言語名は確認できない。

API の形状はエージェント定義用であり、goals と deterministic guardrails の宣言、composable skills (triage、respond、confirm など) の workflows への組み立て、workflow ごとの creativity と determinism の tuning、orchestration、simulations、logic traces による debugging、systems integrations (real-time knowledge、secure action、contact center handoff) である。既存 SDLC (version control、release gating、CI/CD、audit) に適合し、agent は access 可能な Git repository に存在する。汎用 compute ではなく、Agent OS 上で動く agent logic の定義が対象である。ただし context engineering の記事では custom blocks の定義や arbitrary code の記述にも触れており、範囲は agent 定義に沿ったコードに限られる。

参考: https://sierra.ai/product/agent-sdk / https://sierra.ai/blog/agent-development-life-cycle / https://sierra.ai/blog/context-engineering-the-key-to-great-agents

### Q. declarative programming languageということは、TypeScriptやPythonが使えるわけではないのか? 認可のポリシー言語みたいにABAC的な定義ができるのか?

言語名は公開情報では不明である。Clay Bavor は Sequoia のインタビューで purpose built for building agents な declarative programming language と述べており、開発者は what を定義し、how は Agent OS が担うと説明している。existing programming environment で開発するともあるが、TypeScript や Python の対応明言はない。contact center 連携や systems of record 連携の SDKs (複数形) が別にあることにも触れている。

ABAC との analogy は半分正しい。guardrails は deterministic な business rules であり、例として orders can only be returned within 30 days of purchase や、製品全体を語りつつ医療助言は出さない (healthcare-adjacent 事例) がある。attributes や state に対する宣言的制約という点は ABAC の policy に似ており、context engineering の Conditions (認証済み、subscription load 済みなどの state) も属性判定に相当する。違いは、ABAC が subject、resource、action の認可判定であるのに対し、こちらは会話中の agent の振る舞いと action 実行の統制である点と、supervisory agents が live 会話を監視して subtle に修正する runtime 層がある点である。

参考: https://sequoiacap.com/podcast/training-data-clay-bavor / https://sierra.ai/blog/agent-development-life-cycle

### Q. Integration Libraryについて詳しく教えて

Agent Studio 2.0 の機能であり、business systems への接続をノーコード化したものである。3 要素で構成される。Out-of-the-box 40+ は third-party knowledge bases、systems of record、contact centers への pre-built 接続である。具体名の公式一覧はなく、サードパーティ情報では Salesforce、Zendesk、Shopify、Stripe、Twilio、Snowflake、EHR systems が挙がる。Custom integrations は proprietary system 向けであり、opinionated integration framework で Agent Studio 内で設定する。Agent actions は journeys 内で integrations を動作させる使い方であり、顧客代理の real-time action から full context 付きの care team への routing までを指す。

設定手順は library から選択し、credentials と endpoints を追加して publish する。数週間かかった custom 開発が数分になると説明されている。接続すると tools として Agent Studio と Agent SDK の両方に露出し、返品処理、注文更新、account data 取得などに使う。複雑な用途は Agent SDK で拡張できる。プロトコルは MCP、REST、GraphQL、custom に対応し、ChatGPT Apps へ MCP で公開できる。安全性は SoR への deterministic な書き込みと PCI DSS 分離インフラ、Supervisor models で担保する。なお Sierra Interactive は不動産 CRM の別会社であり混同に注意する。

参考: https://sierra.ai/blog/agent-studio-2-0 / https://sierra.ai/product/agent-studio / https://sierra.ai/blog/your-agent-laid-bare-and-why-it-matters

### Q. 「顧客代理の action から care team への routing まで」の意味は? care team とは?

Agent actions の範囲の説明であり、両端の例である。一端はエージェントが顧客の代わりに systems of record へ書き込むこと (返品処理、注文更新など) であり、もう一端は解決できない場合に人間の担当者へ会話を引き継ぐことである。引き継ぎ時は full context (要約など) を付けて渡す。

care team とは人間のカスタマーサポート担当者のチームである。公式の言い換えは care representatives、support teams、human support team、care team member であり、Live Assist はこの人間担当者向けの機能である。

参考: https://sierra.ai/product/agent-studio / https://sierra.ai/product/channels

### Q. Journeysでは実現できず、Agent SDKでは実現できる複雑な要件とは? SDKでどう実現するのか?

公式が挙げる例は subscription churn の管理、regulated workflows の自動化、multi-step sales processes の推進である。Agent OS 2.0 の記事では full control、deep orchestration、rich systems integrations と表現される。Ramp の warranty claim (他システムの guardrails 執行のため画像確認が必要) も SDK 事例である。

実現方法は goals と deterministic guardrails の宣言、composable skills の workflows への組み立て、workflow ごとの tuning、custom blocks と arbitrary code、custom integrations であり、version control、CI/CD、audit の SDLC で管理し、logic traces で debug する。ただし Agent Studio 2.0 で同じ building blocks がノーコード化されており、境界の厳密な機能表は公開情報にない。code 側に残るのは制御の深さと engineering workflow の規模という整理が正確である。

参考: https://sierra.ai/blog/agent-studio-2-0 / https://sierra.ai/blog/agent-os-2-0 / https://sierra.ai/product/agent-sdk

### Q. Custom integrationsは自前実装できるという意味か? Agent SDKによる開発とは別の概念か?

前半は正しい。Custom integrations は proprietary system 向けに自社で接続を作ることであり、opinionated integration framework で Agent Studio 内で設定する。

後半は、別概念ではなく同じ integration の 2 経路である。公式の文言は Integrations are fully extensible through Sierra's Agent SDK, so developers can write and connect their own integrations for more complex use cases である。ノーコードの framework 経路で足りない複雑な用途を SDK のコード経路で作る関係であり、どちらも tools として Agent Studio と Agent SDK の両方に露出する。SDK 自体は integrations 以外 (journeys、goals、guardrails、skills、workflows) も含む広い概念であり、integration 構築はその一面である。

参考: https://sierra.ai/blog/agent-studio-2-0

### Q. 「制御の深さと engineering workflow の規模」とは具体的に何ができることか?

公開情報に厳密な機能表はないため、公式の文言からの具体化である。

制御の深さとは、LLM に判断させない deterministic な処理をコードで書けることである。例として custom blocks と arbitrary code (eligibility 判定、日付計算、record 更新などの厳密規則)、proprietary system 向け custom integrations の実装 (auth、data mapping、error handling)、workflow ごとの creativity と determinism の tuning、複数 systems にまたがる deep orchestration がある。Ramp の warranty claim の画像確認が該当する。

engineering workflow の規模とは、agent をコードとして大人数・大量変更で回せることである。例として Git での branching、code review、diff、一括編集、GitHub Actions での simulation gate と自動 promotion、PR 紐付け snapshots、audit がある。数十 journeys の横断変更や 200 名規模の協調が該当する。

参考: https://sierra.ai/blog/agent-studio-2-0 / https://sierra.ai/blog/agent-os-2-0 / https://sierra.ai/blog/workspaces

### Q. Sierraはワークフローではないと主張している公式文書はどれか?

Context engineering の記事である。Three eras of customer interaction として Era 1: IVR、Era 2: Flow、Era 3: Context engineering を対比し、Flow は predefined path (flowchart、decision tree、digitized SOPs) であり、if this then that で動作し、範囲外は escalate し、SOP 追加で管理困難になると述べる。Era 3 は rigid flows ではなく goals に導かれ guardrails に制約されると主張する。関連文書として Agent Studio 2.0 の記事 (Journeys go beyond sequential workflows and lists of SOPs, which demo well but tend to degrade as they scale) と Agent Development Life Cycle の記事 (prompt engineering による inscrutable な workflows の回避) がある。

参考: https://sierra.ai/blog/context-engineering-the-key-to-great-agents / https://sierra.ai/blog/agent-studio-2-0
