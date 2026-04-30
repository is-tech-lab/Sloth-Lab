# AI-DLC 用語集

## 主要な用語

### フェーズとステージの違い

**フェーズ**: AI-DLC における3つの高レベルなライフサイクルフェーズの1つ
- 🔵 **インセプション・フェーズ (INCEPTION PHASE)** - 計画とアーキテクチャ (WHAT と WHY)
- 🟢 **コンストラクション・フェーズ (CONSTRUCTION PHASE)** - 設計、実装、テスト (HOW)
- 🟡 **オペレーションズ・フェーズ (OPERATIONS PHASE)** - デプロイとモニタリング (将来的な拡張)

**ステージ**: フェーズ内の個別のワークフロー活動
- 例: コンテキスト評価ステージ、要件評価ステージ、コード生成 (Code Generation) ステージ
- 各ステージには特定の前提条件、手順、および出力がある
- ステージは常時実行 (ALWAYS-EXECUTE) または 条件付き (CONDITIONAL) に分類される

**使用例**:
- ✅ "コンストラクション・フェーズ (CONSTRUCTION PHASE) には7つのステージが含まれる"
- ✅ "コード生成 (Code Generation) ステージは常に実行される"
- ✅ "現在はインセプション・フェーズ (INCEPTION PHASE) にあり、要件評価ステージを実行している"
- ❌ "要件評価フェーズ" (正しくは "ステージ")
- ❌ "コンストラクション・ステージ" (正しくは "フェーズ")

## 3フェーズのライフサイクル

### インセプション・フェーズ (INCEPTION PHASE)
**目的**: 計画とアーキテクチャ上の意思決定  
**焦点**: 何を (WHAT)、なぜ (WHY) 構築するかを決定する  
**場所**: `inception/` ディレクトリ

**ステージ**:
- ワークスペース検出 (Workspace Detection) (常時 (ALWAYS))
- リバースエンジニアリング (Reverse Engineering) (条件付き (CONDITIONAL) - ブラウンフィールド (Brownfield) のみ)
- 要件分析 (Requirements Analysis) (常時 (ALWAYS) - 適応的な深度レベル)
- ユーザーストーリー (User Stories) (条件付き (CONDITIONAL))
- ワークフロープランニング (Workflow Planning) (常時 (ALWAYS))
- アプリケーション設計 (Application Design) (条件付き (CONDITIONAL))
- ユニット生成 (Units Generation) (条件付き (CONDITIONAL))

**出力物**: 要件、ユーザーストーリー、アーキテクチャ上の意思決定、ユニット定義

### コンストラクション・フェーズ (CONSTRUCTION PHASE)
**目的**: 詳細設計と実装  
**焦点**: どのように (HOW) 構築するかを決定する  
**場所**: `construction/` ディレクトリ

**ステージ**:
- 機能設計 (Functional Design) (条件付き (CONDITIONAL)、ユニットごと)
- NFR要件 (NFR Requirements) (条件付き (CONDITIONAL)、ユニットごと)
- NFR設計 (NFR Design) (条件付き (CONDITIONAL)、ユニットごと)
- インフラ設計 (Infrastructure Design) (条件付き (CONDITIONAL)、ユニットごと)
- コード生成 (Code Generation) (常時 (ALWAYS)) — パート1: 計画 と パート2: 生成 を含む
- ビルドとテスト (Build and Test) (常時 (ALWAYS))

**出力物**: 設計成果物、NFR実装、コード、テスト

### オペレーションズ・フェーズ (OPERATIONS PHASE)
**目的**: デプロイと運用準備  
**焦点**: デプロイおよび実行方法  
**場所**: `operations/` ディレクトリ

**ステージ**:
- オペレーションズ (Operations) (プレースホルダー (PLACEHOLDER))

**出力物**: ビルド手順、デプロイガイド、モニタリング設定、検証手順

---

## ワークフローステージ

### 常時実行 (ALWAYS-EXECUTE) ステージ
- **ワークスペース検出 (Workspace Detection)**: ワークスペースの状態とプロジェクトタイプの初期分析
- **要件分析 (Requirements Analysis)**: 要件の収集 (深度レベルは複雑さに応じて変化)
- **ワークフロープランニング (Workflow Planning)**: どのフェーズを実行するかの実行計画の作成
- **コード生成 (Code Generation)**: パート1 (計画) で詳細な実装計画を作成し、パート2 (生成) で計画および事前成果物に基づいて実際のコードを生成する、2つのパートから構成される単一ステージ
- **ビルドとテスト (Build and Test)**: 全ユニットのビルドと包括的なテストの実行

### 条件付き (CONDITIONAL) ステージ
- **リバースエンジニアリング (Reverse Engineering)**: 既存コードベースの分析 (ブラウンフィールド (Brownfield) プロジェクトのみ)
- **ユーザーストーリー (User Stories)**: ユーザーストーリーとペルソナの作成 (ストーリー計画とストーリー生成を含む)
- **アプリケーション設計 (Application Design)**: アプリケーションのコンポーネント、メソッド、ビジネスルール、サービスの設計
- **ユニット生成 (Units Generation)**: システムのユニット・オブ・ワーク (unit of work) への分解 (内部的な計画・生成のサブステップと、ユニットごとの設計を含む)
- **機能設計 (Functional Design)**: 技術非依存のビジネスロジック設計 (ユニットごと)
- **NFR要件 (NFR Requirements)**: NFR の決定とテックスタックの選定 (ユニットごと)
- **NFR設計 (NFR Design)**: NFR パターンと論理コンポーネントの組み込み (ユニットごと)
- **インフラ設計 (Infrastructure Design)**: 実際のインフラサービスへのマッピング (ユニットごと)

## アプリケーション設計の用語

- **コンポーネント (Component)**: 特定の責務を持つ機能的なユニット
- **メソッド (Method)**: コンポーネント内の関数または操作で、定義されたビジネスルールを持つ
- **ビジネスルール**: メソッドの動作とバリデーションを制御するロジック
- **サービス (Service)**: コンポーネント間でビジネスロジックを調整するオーケストレーション層
- **コンポーネント依存関係**: コンポーネント間の関係と通信パターン

## アーキテクチャ用語 (インフラ)

### ユニット・オブ・ワーク (Unit of Work)
開発目的でユーザーストーリーを論理的にグループ化したもの。計画と分解の際に使用する用語。

**使用例**: "システムをユニット・オブ・ワーク (unit of work) に分解する必要がある"

### サービス (Service)
マイクロサービスアーキテクチャにおける独立してデプロイ可能なコンポーネント。各サービスは個別のユニット・オブ・ワーク (unit of work) である。

**使用例**: "Payment サービス (Service) はすべての決済処理を担当する"

### モジュール (Module)
単一サービスまたはモノリス内の機能の論理的なグループ化。モジュール (Module) は独立してデプロイできない。

**使用例**: "User サービス (Service) 内の認証モジュール (Module)"

### コンポーネント (Component)
サービスまたはモジュール (Module) 内の再利用可能なビルディングブロック。コンポーネント (Component) は特定の機能を提供するクラス、関数、またはパッケージである。

**使用例**: "EmailValidator コンポーネント (Component) はメールアドレスを検証する"

## 用語使用ガイドライン

### 各用語の使用場面

**ユニット・オブ・ワーク (Unit of Work)**:
- ユニット生成 (Units Generation) ステージ中
- システム分解について議論する際
- 計画文書や議論の中で
- 例: "これをユニット・オブ・ワーク (unit of work) にどう分解すべきか？"

**サービス (Service)**:
- 独立してデプロイ可能なコンポーネントを指す場合
- マイクロサービスアーキテクチャの文脈で
- デプロイおよびインフラの議論の中で
- 例: "Order サービス (Service) は ECS にデプロイされる"

**モジュール (Module)**:
- サービス内の論理的なグループ化を指す場合
- モノリスアーキテクチャの文脈で
- 内部構造について議論する際
- 例: "レポートモジュール (Module) はすべてのレポートを生成する"

**コンポーネント (Component)**:
- 特定のクラス、関数、またはパッケージを指す場合
- 設計と実装の議論の中で
- 再利用可能なビルディングブロックについて議論する際
- 例: "DatabaseConnection コンポーネント (Component) は接続を管理する"

## ステージの用語

### 計画 (Planning) vs 生成 (Generation)
- **計画 (Planning)**: 実行のための質問とチェックボックスを含むプランの作成
- **生成 (Generation)**: プランを実行して成果物を作成すること

例 (これらは個別のステージではなく、単一ステージ内の内部サブステップ):
- ストーリー計画 → ストーリー生成 (ユーザーストーリー (User Stories) ステージ内)
- ユニット計画 → ユニット生成 (ユニット生成 (Units Generation) ステージ内)
- ユニット設計計画 → ユニット設計生成 (ユニットごとの設計内)
- NFR計画 → NFR生成 (NFR要件 (NFR Requirements) ステージ内)
- コード生成 (Code Generation) パート1 (計画) → コード生成 (Code Generation) パート2 (生成)

### 深度レベル (Depth Levels)
- **最小 (Minimal)**: シンプルな変更のための迅速かつ集中した実行
- **標準 (Standard)**: 典型的なプロジェクト向けの標準的な成果物を伴う通常の深度
- **包括的 (Comprehensive)**: 複雑・高リスクなプロジェクト向けの全成果物を伴う最大深度

## 成果物の種類

### プラン (Plans)
実行を導くチェックボックスと質問を含むドキュメント。
- `aidlc-docs/plans/` に格納
- 例: `story-generation-plan.md`、`unit-of-work-plan.md`

### 成果物 (Artifacts)
プランを実行することで生成される出力物。
- `aidlc-docs/` の各サブディレクトリに格納
- 例: `requirements.md`、`stories.md`、`design.md`

### 状態ファイル (State Files)
ワークフローの進捗と状態を追跡するファイル。
- `aidlc-state.md`: ワークフロー全体の状態
- `audit.md`: すべてのやり取りの完全な監査証跡

## よく使われる略語

- **AI-DLC**: AI-Driven Development Life Cycle (AI 駆動開発ライフサイクル)
- **NFR**: Non-Functional Requirements (非機能要件)
- **UOW**: Unit of Work (ユニット・オブ・ワーク)
- **API**: Application Programming Interface (アプリケーションプログラミングインタフェース)
- **CDK**: Cloud Development Kit (AWS)
