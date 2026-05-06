# Unit of Work Plan — Refactor the World (RTW) MVP

## 概要

RTW MVPを4つのユニット・オブ・ワークに分解し、Construction Phaseの開発ループに対応させます。

**確定ユニット**:
| # | ユニット名 | 種別 |
|---|-----------|------|
| 1 | Backend API | 独立デプロイ可能サービス（ECS Fargate） |
| 2 | AI Integration Service | 独立デプロイ可能サービス（ECS Fargate） |
| 3 | Mobile App | iOSアプリ（React Native / Expo） |
| 4 | AWS Infrastructure | インフラ構成（IaC） |

---

## 実行チェックリスト

### フェーズ1: プランニング準備
- [x] ユニット計画質問の回答収集
- [x] 回答の曖昧さ分析（矛盾・曖昧さなし）
- [x] 分解方針の確定（モノレポ / Backend→AI→Mobile→Infra順 / Mobile主担当 / shared package / AWS CDK）

### フェーズ2: ユニット成果物の生成
- [x] `unit-of-work.md` を生成（ユニット定義・責務・コード整理戦略）
- [x] `unit-of-work-dependency.md` を生成（依存関係マトリクス・開発順序）
- [x] `unit-of-work-story-map.md` を生成（US-01〜US-13 × 4ユニット対応表）
- [x] ユニット境界と依存関係の検証（循環依存なし）
- [x] 全ストーリーがいずれかのユニットに割り当てられていることを確認（13/13 ✓）
- [x] `aidlc-state.md` の進捗更新

---

## ユニット計画質問

以下の質問に回答してください。`[Answer]:` タグの後にアルファベットの選択肢を記入してください。
どの選択肢も当てはまらない場合は最後の選択肢（Other）を選び、希望を記述してください。

---

## 質問 1: リポジトリ・コード整理戦略

4ユニットのコードをどのように管理しますか？（Greenfield必須質問）

A) **モノレポ（pnpm workspaces）** — 1つのGitリポジトリに全ユニットを配置
   ```
   /
   ├── apps/
   │   ├── backend-api/       # Unit 1: Node.js/TypeScript
   │   ├── ai-service/        # Unit 2: Node.js/TypeScript
   │   └── mobile/            # Unit 3: React Native/Expo
   ├── packages/
   │   └── shared/            # 共有型定義・ユーティリティ
   └── infra/                 # Unit 4: IaC（CDK/Terraform）
   ```
   - メリット: 共有型定義の管理が楽。1つのPRで複数ユニットを横断的に変更できる

B) **マルチリポジトリ** — ユニットごとに独立したGitリポジトリ
   ```
   rtw-backend-api/      # Unit 1
   rtw-ai-service/       # Unit 2
   rtw-mobile/           # Unit 3
   rtw-infrastructure/   # Unit 4
   ```
   - メリット: ユニット間の独立性が高い。別チームへの委譲が容易

C) **ハイブリッド（アプリはモノレポ、インフラは別）** — backend/ai/mobileを1リポジトリ、infraを別リポジトリ

D) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 2: ユニット開発順序（Construction Phase のループ順）

Construction Phaseでユニットを開発する順番を選んでください。
AI-DLCは1ユニットずつ順番にFunctional Design → Code Generationをループします。

A) **Backend API → AI Integration Service → Mobile App → AWS Infrastructure**
   - バックエンドAPIを先に実装してAPIスペックを確定 → AI Serviceを実装 → Mobileを最後
   - インフラは最後にまとめて定義（開発中はローカル環境を使う）

B) **AWS Infrastructure → Backend API → AI Integration Service → Mobile App**
   - インフラを先に確定してから実装。クラウドファースト
   - 本番環境での動作確認を早めに行いたい場合に有効

C) **Backend API + AI Integration Service（同時）→ Mobile App → AWS Infrastructure**
   - BackendとAI Serviceは密接に連携するため並行開発。ただしAI-DLCのループは1ユニットずつなので、どちらを先にするか定義は必要

D) Other（AIサービスを先にしたい、Mobileを先にしたい等）

[Answer]: A

---

## 質問 3: クロスカッティングストーリーの主担当ユニット

US-06（AI変換）やUS-09（フィード）のように複数ユニットにまたがるストーリーについて、`unit-of-work-story-map.md` での主担当はどう割り当てますか？

A) **Mobile App を主担当（ユーザー体験フロー基準）** — ユーザーが直接触れる画面の実装として Mobile に帰属。Backend API と AI Service は「Mobile が使う内部API」として各ユニットに実装タスクを持つ
   - ストーリーマップ: US-06「AI変換」= **Mobile App** ← Backend API ← AI Service の順で依存

B) **バックエンド側を主担当（機能ロジック基準）** — コアロジックを持つユニットに帰属。UI実装は副次的に Mobile に割り当て
   - ストーリーマップ: US-06「AI変換」= **AI Service**（パイプライン）+ Backend API（API）+ Mobile（UI）として個別割当

C) **全ユニットに分割割当** — クロスカッティングストーリーは各ユニットへ部分的に分割して割り当てる（例: US-06-A: AI変換パイプライン実装 → AI Service, US-06-B: /transform API実装 → Backend API, US-06-C: 変換UI実装 → Mobile App）

D) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 4: 共有TypeScript型定義の戦略

Backend API と AI Integration Service（および共有部分）でTypeScript型をどう管理しますか？

A) **共有パッケージ（packages/shared）** — モノレポ前提。PostOutput, TransformInput/Output など共通の型を shared パッケージに定義。各ユニットがimport
   - Q1でAを選んだ場合に最も自然

B) **各ユニットで独自定義（型の重複を許容）** — シンプル。ユニット間の型の同期は手動
   - マルチリポジトリの場合、または型の共有が少ない場合に有効

C) **OpenAPI / スキーマ定義から型を生成** — API定義（OpenAPI spec）を基準として、各ユニットの型を自動生成
   - ドキュメントと型の一貫性が保てる

D) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A

---

## 質問 5: AWS InfrastructureユニットのIaC戦略

インフラのコード化（Infrastructure as Code）について選んでください。

A) **AWS CDK（TypeScript）** — Node.js/TypeScriptでインフラ定義。Backend API・AI Serviceと同じ言語。スタック定義がコードとして表現できる
   - RTWのバックエンドスタックと同じ言語なので一貫性が高い

B) **Terraform** — HCL言語によるインフラ定義。クラウド非依存な記述が可能
   - チームにTerraform経験者がいる場合に有効

C) **AWS CloudFormation（YAML/JSON）** — AWSネイティブ。CDKのコンパイル先でもある
   - 低レベルで冗長だが、AWSとの直接統合

D) **IaCは今回のスコープ外（手動構築）** — Construction PhaseではIaCを書かず、インフラは手動構築の手順書のみを作成

E) Other (その他の場合は [Answer]: タグの後に記述してください)

[Answer]: A
