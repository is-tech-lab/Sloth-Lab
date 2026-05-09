# Sloth Feed — `Sloth-Lab`

> ハッカソンBチーム：神津、蓜島、林、村田

---

## 🦥 一行紹介

**「『仕事じゃないけど、、、』が世の中を変える」**
— AI ナマケモノが、Larry Wall・ラッセル・老子の系譜から、あなたの怠惰そのものを肯定する**動的IPプロダクト**。

---

## ✍️ Intent（事業意図・1段落）

Sloth Feed は、**「人をダメにする」を旗印とする動的IP プロダクト**である。AI ナマケモノが Larry Wall・ラッセル・老子の系譜から**ユーザーの怠惰そのものを肯定する**動的応答を返し、罪悪感ループから人々を解放する。SNS は IP の入口・コミュニティ装置であり、本業は IP の育成（物販・ライセンス・コラボ・出版）である。

**ビジョン**：『仕事じゃないけど、、、』が世の中を変える
**副スローガン**：ダメだけど、世の中を変える

---

## 🎯 テーマ「人をダメにする」との適合

ハッカソンテーマ「人をダメにする」（≒ エンジニア三大美徳の系譜）に、Sloth Feed は**プロダクトの全レイヤーで応える**：

| レイヤー | 「人をダメにする」の体現 |
|---|---|
| キャラクター | **ナマケモノ**（Sloth = なまけもの） |
| パンチライン | 『仕事じゃないけど、、、』が世の中を変える |
| 応答する AI 人格 | **達観した怠惰の老師**（80歳の元学者・拒絶せず・比較せず・断定する） |
| 投稿対象 | **怠惰系**（布団から3時間出られない・二度寝・録画一気見）と**小さな善行系**（家族にLINE・洗い物・植物に水やり）を**等しく**受容 |
| 弾くもの | 仕事の成果・旅行・キラキラ充実投稿 |
| UI | **サンドイッチUI**（上「仕事じゃないけど」/ 中：投稿+AIコメント / 下「これが世の中を変える」） |
| 倫理 | **ファンとして遇する**（toBデータ販売・加工レポート・広告 すべて完全禁止）|

---

## 🛣️ 5経路フレーム — ダメが世の中を変えるメカニズム

「それは偉業だ」と褒めるのではなく、**何が・どう・世の中を変えるのか**を AI ナマケモノが具体的に語る。投稿内容に応じて、いずれかの経路に紐付けて応答する：

1. **過剰生産社会へのブレーキ** — 24時間生産命令への小さな抵抗
2. **創造の余白の保持** — DMN（デフォルト・モード・ネットワーク）こそ AI に代替されない人間性
3. **多様性の保護** — 「頑張る一強」社会への選択肢を維持
4. **自己への暴力の停止** — 罪悪感ループは社会的損失。自分を許すことが社会全体の精神的健康に寄与
5. **集積による文化変容** — 一人の二度寝は事件ではないが、500万人の二度寝は労働政策を動かす

---

## 🏗️ 技術構成（PoC スコープ）

| レイヤー | 採用技術 |
|---|---|
| フロント・サーバ | **Next.js 14+ App Router / TypeScript** |
| 認証 | **Auth.js (NextAuth v5) + AWS Cognito User Pool**（OAuth/OIDC・HttpOnly Cookie・JWT） |
| AI 呼び出し | **Amazon Bedrock 経由の Claude**（`@aws-sdk/client-bedrock-runtime`、IAM 認証） |
| データストア | **AWS DynamoDB**（Posts のみ。Users は Cognito 一本化で PoC 外） |
| 引用ソース戦略 | **PoC：LLM の学習済み知識を信用**（RAG 不採用）／**Phase 2：S3 + Agentic Search** で事実検証（案・未合意） |
| AI 機能 | フィルタリング（仕事系除外）／5経路紐付け生成／個別化記憶（UserHistory）／依存防止（切り上げ提案） |

---

## 🧱 ユニット分解（3 ユニット・9 ストーリー）

| Unit | 担当 | ストーリー | 本数 |
|---|---|---|---|
| **Unit 1** | Auth + IPファン識別基盤（Cognito + Auth.js セットアップ + 共通基盤） | US-001, US-002 | 2 |
| **Unit 2** | **ナマケモノ対話エンジン（動的IPの核）**：5経路 + 老師人格 + 個別化記憶 + 依存防止 | US-003〜006, US-009 | **5（最多）** |
| **Unit 3** | 共同体タイムライン（サンドイッチUI） | US-007, US-008 | 2 |

依存：Unit 1 → Unit 2 → Unit 3
**Unit 2+3 は当初分離していたが、サンドイッチUI が Unit 2 で生成された AI コメントを前提とする強結合のため統合**（経緯は [`unit-of-work.md`](aidlc-docs/inception/application-design/unit-of-work.md) の統合経緯セクション）。

---

## 📂 ドキュメント構造

```
Sloth-Lab/
├── README.md                                  ← このファイル（審査員向け1ページ要約）
├── docs/
│   └── ideation/
│       ├── customer_insights.md              ← ペルソナ詳細
│       ├── ideas.md                          ← コンセプト発想・競合分析・4差別化軸
│       └── commercialization.md              ← IP 事業マニフェスト・5レイヤー収益モデル
├── mock/
│   └── index.html                            ← サンドイッチUI ビジュアルモック
└── aidlc-docs/                               ← AI-DLC 成果物
    ├── aidlc-state.md                        ← フェーズ進行状態（3周分）
    ├── audit.md                              ← 全インタラクション監査ログ
    └── inception/
        ├── project-overview.md               ← Intent / ビジョン / 5経路 / 4差別化軸
        ├── requirements/
        │   ├── requirements.md               ← FR-001〜011 / NFR-001〜008
        │   └── requirement-verification-questions.md
        ├── user-stories/
        │   ├── personas.md                   ← ユイ（24歳）/ ケンタ（28歳）
        │   └── stories.md                    ← US-001〜009（達観した怠惰の老師受け入れ基準付き）
        ├── application-design/
        │   ├── application-design.md         ← アーキテクチャ概要
        │   ├── components.md                 ← 主要コンポーネント
        │   ├── component-methods.md          ← メソッド・型定義（CreatePostResult enum 等）
        │   ├── component-dependency.md       ← 依存マトリクス・データフロー
        │   ├── services.md                   ← サービス層
        │   ├── unit-of-work.md               ← 3 ユニット定義・統合経緯
        │   ├── unit-of-work-dependency.md    ← ユニット間依存・共有リソース
        │   ├── unit-of-work-story-map.md     ← US-001〜009 マッピング
        │   ├── security-review.md            ← 技術選択のセキュリティレビュー（OWASP Top 10）
        │   └── version-management-review.md  ← バージョン管理レビュー（2024〜2025 インシデント対応）
        └── plans/                            ← AI-DLC ステージごとの計画ファイル
            ├── execution-plan.md             ← 全体フローチャート（Mermaid）
            ├── application-design-plan.md
            ├── story-generation-plan.md
            ├── unit-of-work-plan.md
            └── user-stories-assessment.md
```

---

## 🔄 AI-DLC サイクル履歴（3 周）

| 周回 | 日付 | 主な変更 |
|---|---|---|
| 1周目 | 2026-05-07 | Greenfield で初期構築。3ユニット（Auth / Post+AI / Feed）確定 |
| 2周目 | 2026-05-09 | [Issue #5](https://github.com/is-tech-lab/Sloth-Lab/issues/5) の帰着を反映。SNS事業 → IP事業へ転換、toBデータ販売完全放棄、動的IP × AI技術の三位一体に再構成 |
| 3周目 | 2026-05-09 | 各ステージで人間レビュー実施。Bedrock 経由 Claude に統一・RAG 除外（Phase 2 で S3 + Agentic Search）・「達観した怠惰の老師」人格・🦥はヘッダ限定・5経路ラベル受け入れ基準・US-009（依存防止）追加・**Auth.js + AWS Cognito 全面切替**・PR Review 12 項目すべて解消 |

詳細は [`aidlc-state.md`](aidlc-docs/aidlc-state.md) と [`audit.md`](aidlc-docs/audit.md) を参照。

---

## 🚦 PoC vs Phase 2（境界）

- **PoC**：INCEPTION + CONSTRUCTION 完了まで。Cognito + Auth.js / Bedrock Claude / DynamoDB Posts / 学習済み知識による引用。
- **Phase 2**（**案・未合意**）：S3 + Agentic Search による引用事実検証、5レイヤー収益モデルの段階導入、IP 物販・出版・コラボ展開。

---

## 🔗 関連 Issue / PR

- [Issue #5（Closed）](https://github.com/is-tech-lab/Sloth-Lab/issues/5) — toBデータ販売の構造的自己矛盾と帰着
- [Issue #7](https://github.com/is-tech-lab/Sloth-Lab/issues/7) — Issue #5 帰着の反映 ― ideation 修正 + AI-DLC 再実行
- [PR #8 / PR #9](https://github.com/is-tech-lab/Sloth-Lab/pulls) — 2 周目 / 3 周目の成果物反映

---

## 🏆 審査観点との対応（自己評価）

| 観点 | 主な根拠 |
|---|---|
| **ビジネス意図（Intent）の明確さ** | 上記 Intent 段落 / 5レイヤー収益モデル / [`commercialization.md`](docs/ideation/commercialization.md) |
| **創造性とテーマ適合性** | 5経路フレーム / 達観した怠惰の老師人格 / サンドイッチUI / 怠惰・善行両受容 / 文化アンカー（Larry Wall・ラッセル・老子・ニュートン・フレミング・ケインズ） |
| **Unit 分解の適切さ** | 3 ユニット構成 / Unit 2+3 統合経緯の文書内自己完結 / 依存マトリクス / 9 ストーリー全カバー |
| **ドキュメントの品質** | 3 周分の進化履歴追跡可能 / 内部相互参照 / PR Review 12 項目解消 / Auth.js 移行 10 文書カスケード更新 / Mermaid 可視化 |
| **アイデアと技術のバランス** | 思想（5経路 + 老師人格 + ファン原則）× 技術（Auth.js + Cognito + Bedrock + DynamoDB + UserHistory）が橋渡しされている |
