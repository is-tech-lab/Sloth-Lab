# AI-DLC State Tracking

## Project Information
- **Project Name**: Sloth Feed
- **Project Type**: Greenfield
- **Start Date**: 2026-05-07T00:00:00Z
- **Current Stage**: INCEPTION PHASE 完了（3回目サイクル完了・2026-05-09）

## Workspace State
- **Existing Code**: No
- **Reverse Engineering Needed**: No
- **Workspace Root**: /Users/user/repos/Sloth-Lab

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## 📌 CONSTRUCTION フェーズ着手時の必読事項（ピン留め・2026-05-09）

CONSTRUCTION フェーズに進む際は、以下 2 文書を**実装計画策定の前に必ず参照**すること。Code Generation のチェックリストや CI 設定の出発点として組み込む：

1. **[セキュリティレビュー](inception/application-design/security-review.md)** — OWASP Top 10 観点・実装時 Code Review チェックリスト 11 項目（middleware matcher / `await auth()` 二重ガード / `AUTH_SECRET` 32 文字 / Bedrock IAM `InvokeModel` のみ / Prompt Injection 5 対策 等）
2. **[バージョン管理レビュー](inception/application-design/version-management-review.md)** — 2024〜2025 インシデント対応・実装着手前に決定すべき 3 点（**Next.js 14.2.25+ / 15.2.3+ 必須（CVE-2025-29927）** / **AWS クレデンシャルは IAM ロール一本化（shai-hulud）** / **CI で `npm ci` + `npm audit --audit-level=high`（chalk-debug）**）+ PoC ベースライン 13 項目

CONSTRUCTION の最初のステージ（Functional Design 〜 Code Generation）で、**両文書のチェックリストを実装タスクへ機械的に展開する**こと。

---

## Project Frame（2026-05-09 更新）
- **事業フレーム**: IP事業 × 動的IP × AI技術
- **コアコンセプト**: 「人をダメにする」運動
- **ビジョン**: 『仕事じゃないけど、、、』が世の中を変える
- **副スローガン**: ダメだけど、世の中を変える
- **AI 呼び出し基盤**: Amazon Bedrock 経由の Claude（`@aws-sdk/client-bedrock-runtime`、IAM 認証）
- **引用ソース戦略**: PoC は LLM の学習済み知識を信用（RAG 不採用）／Phase 2 で S3 + Agentic Search 実装予定
- **認証スタック**: Auth.js (NextAuth v5) + AWS Cognito User Pool（OAuth/OIDC）／HttpOnly Cookie 保存／登録 UI は PoC 実装時に決定（Hosted UI vs 自前）
- **データ販売**: 完全禁止（toBデータ販売・加工レポート販売・広告掲載すべて）

## Extension Configuration
| Extension | Enabled | Decided At |
|-----------|---------|------------|
| Security Baseline | No | Requirements Analysis |
| Property-Based Testing | No | Requirements Analysis |

## Stage Progress

### INCEPTION PHASE（3回目: 2026-05-09 開始・正式再構成・各ステージで人間レビュー）

- [x] Stage 1: Workspace Detection — Greenfield 確認、既存 aidlc-state.md／2回目サイクルまでの成果物確認
- [x] Stage 2: Requirements Analysis — Issue 1〜4修正＋FR-009/010/011・NFR-007/008追加・成功基準再構築。技術スタックを Bedrock 経由に統一・RAG除外（Phase 2 で S3 + Agentic Search）・FR-010 を「達観した怠惰の老師」人格に再設計
- [x] Stage 3: User Stories — Issue 1〜7 修正（用語統一／ハルシネーション記述／経路ラベル受け入れ基準／新US-009 依存防止／老師人格制約／🦥はヘッダ限定／残置「ダメ」用語の整合）
- [x] Stage 4: Workflow Planning — Issue 1〜6 修正（Unit分解 4→3整合／3周サイクル位置づけ／変更影響評価・リスク評価・成功基準の新版整合／Mermaid化／Phase 2構想セクション「案・未合意」明記／履歴4ファイルに位置づけ注記）。**用語明確化**（PoC = INCEPTION + CONSTRUCTION、Phase 2 = PoC 完成後、S3 + Agentic Search は Phase 2）
- [x] Stage 5: Application Design — Issue 1〜3 + Issue 2-A 修正（components.md / component-methods.md / component-dependency.md：AICommentService → AINamakemonoService 改名、BrandFrame 追加、5経路紐付け責務、個別化記憶連携、UserHistory 新規、Post 型に authorName/pathway 追加、CreatePostResult を部分失敗別 enum に拡張、JWT 保存方針明記、ページネーション PoC=50件上限、バリデーション要件明記、依存マトリクス更新、データフロー図全面再構築、DynamoDB スキーマに name/authorName/aiCitationSource/pathway 追加）
- [x] Stage 6: Units Generation — Issue 1〜3 修正（unit-of-work.md 用語整合・新FR-009/010/011反映・完了基準再構築 / unit-of-work-dependency.md に Unit 2→UserRepository 依存追加・UserHistory/BrandFrame 共有リソース化 / unit-of-work-story-map.md 全面書き直し US-001〜009 全 9 ストーリーを 3 ユニットにマッピング）
- [x] Stage 6 PR Review 対応 — PR #8 コメント照合（C-1/C-2/W-1/W-3/W-5/M-1/M-2/M-4/M-C は Stage 1〜5 で解消済 / W-2: Unit 2 統合理由 / W-4: DynamoDB Scan 注記 / M-3: errors.ts インターフェース / M-5: app/layout.tsx）追加修正5件適用
- [x] Stage 6 Auth.js 移行 — PR W-5 / M-A/B 議論を通じて **Auth.js (NextAuth v5) + AWS Cognito User Pool** へ全面切替を決定。10文書をカスケード更新（component-methods.md / requirements.md / components.md / component-dependency.md / services.md / application-design.md / unit-of-work.md / unit-of-work-dependency.md / unit-of-work-story-map.md / stories.md）。自前 AuthService / UserRepository / bcrypt / JWT 廃止、HttpOnly Cookie / Cognito 一本化、Users テーブル PoC 外、UI 形態（Hosted UI/自前）は PoC 実装時に決定
- [x] Stage 7: 完了処理（2026-05-09）— 審査観点 4 軸（Intent明確さ / 創造性とテーマ適合性 / Unit分解の適切さ / ドキュメントの品質）+ アイデアと技術のバランスを最終照合。`README.md` に審査員向け 1 ページ要約（154 行）を作成し、すべての成果物への導線・3周のサイクル履歴・PoC vs Phase 2 境界・審査観点との対応を集約。INCEPTION PHASE（3回目）正式完了マーク。CONSTRUCTION PHASE はユーザー判断で SKIP 継続

### INCEPTION PHASE（1回目: 2026-05-07）
- [x] Workspace Detection — Greenfield確認、コード未存在
- [x] Requirements Analysis — 要件定義完了
- [x] User Stories — personas.md・stories.md生成完了
- [x] Workflow Planning — 実行計画確定
- [x] Application Design — 設計成果物生成完了
- [x] Units Generation — 3ユニット確定（Auth / Post+AI / Feed）

### INCEPTION PHASE（2回目: 2026-05-09・Issue #5 帰着反映）
- [x] ideation 修正（Phase 1）
  - [x] customer_insights.md：ペルソナを「ダメ全振り」前提に再構築
  - [x] ideas.md：競合分析と4つの差別化軸を追加
  - [x] commercialization.md：SNS事業 → IP事業へ全面書き換え
  - [x] project-overview.md：動的IP × AI技術の三位一体を中核に据える
- [x] Requirements Analysis 再実行 — FR-006（個別化記憶）、FR-007（RAG引用ライブラリ）、FR-008（サンドイッチUI）追加
- [x] User Stories 再考 — 「ダメ全振り」前提で再構築、US-006（関係性継続）追加（全8ストーリー）
- [x] Application Design 意味的再記述 — 3ユニットの構造変更なし、責務を「動的IP × AI」文脈で再定義
- [x] 不整合解消 — Posts.stamps 削除、aiCitationSource 追加、workspace root 修正

### CONSTRUCTION PHASE
- [ ] Functional Design — SKIPPED（ユーザー判断）
- [ ] NFR Requirements — SKIPPED（ユーザー判断）
- [ ] NFR Design — SKIPPED（ユーザー判断）
- [ ] Infrastructure Design — SKIPPED（ユーザー判断）
- [ ] Code Generation — SKIPPED（ユーザー判断）
- [ ] Build and Test — SKIPPED（ユーザー判断）

## Related Issues / PRs
- [Issue #5（Closed）](https://github.com/is-tech-lab/Sloth-Lab/issues/5) — toBデータ販売の構造的自己矛盾と帰着
- [Issue #7](https://github.com/is-tech-lab/Sloth-Lab/issues/7) — Issue #5帰着の反映 ― ideation 修正 + AI-DLC 再実行
