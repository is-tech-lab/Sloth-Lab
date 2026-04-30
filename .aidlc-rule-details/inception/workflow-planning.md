# ワークフロープランニング (Workflow Planning)

**目的**: 実行するフェーズを決定し、包括的な実行計画を作成する

**常時実行 (Always Execute)**: このフェーズは要件とスコープを理解した後、常に実行される

## ステップ 1: すべての過去コンテキストの読み込み

### 1.1 リバースエンジニアリング成果物の読み込み（ブラウンフィールド (Brownfield) の場合）
- architecture.md
- component-inventory.md
- technology-stack.md
- dependencies.md

### 1.2 要件分析の読み込み
- requirements.md（意図分析を含む）
- requirement-verification-questions.md（回答付き）

### 1.3 ユーザーストーリーの読み込み（実行された場合）
- stories.md
- personas.md

## ステップ 2: 詳細なスコープと影響の分析

**完全なコンテキスト（要件 + ストーリー）が揃ったので、詳細な分析を実施する:**

### 2.1 変換スコープの検出（ブラウンフィールド (Brownfield) のみ）

**ブラウンフィールド (Brownfield) プロジェクトの場合**、変換スコープを分析する:

#### アーキテクチャの変換
- **単一コンポーネントの変更** vs **アーキテクチャの変換**
- **インフラの変更** vs **アプリケーションの変更**
- **デプロイメントモデルの変更**（Lambda→コンテナ、EC2→サーバーレスなど）

#### 関連コンポーネントの識別
変換の場合、以下を識別する:
- 更新が必要な**インフラコード**
- 変更が必要な **CDK スタック**
- **API Gateway** 設定
- **ロードバランサー**の要件
- 必要な**ネットワーキング**の変更
- **監視/ロギング**の適応

#### クロスパッケージの影響
- 更新が必要な **CDK インフラ**パッケージ
- バージョン更新が必要な**共有モデル**
- エンドポイント変更が必要な**クライアントライブラリ**
- 新しいテストシナリオが必要な**テストパッケージ**

### 2.2 変更影響評価

#### 影響領域
1. **ユーザー向けの変更**: ユーザーエクスペリエンスに影響するか？
2. **構造的な変更**: システムアーキテクチャが変わるか？
3. **データモデルの変更**: データベーススキーマやデータ構造に影響するか？
4. **API の変更**: インターフェースやコントラクトに影響するか？
5. **NFR への影響**: パフォーマンス、セキュリティ、またはスケーラビリティに影響するか？

#### アプリケーションレイヤーへの影響（該当する場合）
- **コードの変更**: 新しいエントリーポイント、アダプター、設定
- **依存関係**: 新しいライブラリ、フレームワークの変更
- **設定**: 環境変数、設定ファイル
- **テスト**: ユニットテスト、統合テスト

#### インフラレイヤーへの影響（該当する場合）
- **デプロイメントモデル**: Lambda→ECS、EC2→Fargate など
- **ネットワーキング**: VPC、セキュリティグループ、ロードバランサー
- **ストレージ**: 永続ボリューム、共有ストレージ
- **スケーリング**: オートスケーリングポリシー、キャパシティプランニング

#### オペレーションズレイヤーへの影響（該当する場合）
- **監視**: CloudWatch、カスタムメトリクス、ダッシュボード
- **ロギング**: ログ集約、構造化ロギング
- **アラート**: アラーム設定、通知チャンネル
- **デプロイメント**: CI/CD パイプラインの変更、ロールバック戦略

### 2.3 コンポーネント関係マッピング（ブラウンフィールド (Brownfield) のみ）

**ブラウンフィールド (Brownfield) プロジェクトの場合**、コンポーネント依存関係グラフを作成する:

```markdown
## Component Relationships
- **Primary Component**: [Package being changed]
- **Infrastructure Components**: [CDK/Terraform packages]
- **Shared Components**: [Models, utilities, clients]
- **Dependent Components**: [Services that call this component]
- **Supporting Components**: [Monitoring, logging, deployment]
```

各関連コンポーネントについて:
- **変更タイプ**: 主要、マイナー、設定のみ
- **変更理由**: 直接依存、デプロイメントモデル、ネットワーキング
- **変更優先度**: 重要、重要、任意

### 2.4 リスク評価

リスクレベルを評価する:
1. **低 (Low)**: 孤立した変更、容易なロールバック、十分に理解されている
2. **中 (Medium)**: 複数のコンポーネント、中程度のロールバック、一部の不確実性
3. **高 (High)**: システム全体への影響、複雑なロールバック、重大な不確実性
4. **重大 (Critical)**: 本番重要システム、困難なロールバック、高い不確実性

## ステップ 3: フェーズの決定

### 3.1 ユーザーストーリー — 実行済みかスキップか？
**実行済み**: 次の決定へ移る
**未実行 — 以下の場合に実行する**:
- 複数のユーザーペルソナがいる
- ユーザーエクスペリエンスへの影響がある
- 受け入れ基準が必要
- チームのコラボレーションが必要

**以下の場合にスキップする**:
- 内部リファクタリング
- 再現方法が明確なバグ修正
- 技術的負債の削減
- インフラの変更

### 3.2 アプリケーション設計 (Application Design) — 以下の場合に実行する:
- 新しいコンポーネントまたはサービスが必要
- コンポーネントメソッドとビジネスルールの定義が必要
- サービスレイヤー設計が必要
- コンポーネント依存関係の明確化が必要

**以下の場合にスキップする**:
- 既存のコンポーネント境界内での変更
- 新しいコンポーネントまたはメソッドなし
- 純粋な実装変更

### 3.3 ユニット生成 (Units Generation) — 以下の場合に実行する:
- 新しいデータモデルまたはスキーマ
- API の変更または新しいエンドポイント
- 複雑なアルゴリズムまたはビジネスロジック
- 状態管理の変更
- 複数のパッケージで変更が必要
- Infrastructure-as-Code の更新が必要

**以下の場合にスキップする**:
- シンプルなロジックの変更
- UI のみの変更
- 設定の更新
- 簡単な実装

### 3.4 NFR 実装 — 以下の場合に実行する:
- パフォーマンス要件がある
- セキュリティ上の考慮事項がある
- スケーラビリティの懸念がある
- 監視/オブザーバビリティが必要

**以下の場合にスキップする**:
- 既存の NFR セットアップで十分
- 新しい NFR 要件がない
- NFR への影響がないシンプルな変更

## ステップ 4: 適応的な詳細に注意する

**適応深度の説明については [depth-levels.md](../common/depth-levels.md) を参照**

実行される各ステージについて:
- 定義されたすべての成果物が作成される
- 成果物内の詳細レベルは問題の複雑性に適応する
- モデルは問題の特性に基づいて適切な詳細を決定する

## ステップ 5: マルチモジュール調整分析（ブラウンフィールド (Brownfield) のみ）

**複数のモジュール/パッケージを持つブラウンフィールド (Brownfield) の場合**、依存関係を分析し、最適な更新戦略を決定する:

### 5.1 モジュール依存関係の分析
- ビルドシステムの依存関係と依存関係マニフェストを調べる
- ビルド時 vs ランタイムの依存関係を識別する
- モジュール間の API コントラクトと共有インターフェースをマッピングする

### 5.2 更新戦略の決定
依存関係分析に基づいて決定する:
- **更新シーケンス**: 依存関係のため最初に更新する必要があるモジュール
- **並列化の機会**: 同時に更新できるモジュール
- **調整要件**: バージョン互換性、API コントラクト、デプロイ順序
- **テスト戦略**: モジュールごとのテスト vs 統合テストのアプローチ
- **ロールバック戦略**: シーケンス途中で障害が発生した場合の復旧計画

### 5.3 調整計画の文書化
```markdown
## Module Update Strategy
- **Update Approach**: [Sequential/Parallel/Hybrid]
- **Critical Path**: [Modules that block other updates]
- **Coordination Points**: [Shared APIs, infrastructure, data contracts]
- **Testing Checkpoints**: [When to validate integration]
```

影響を受ける各モジュールについて識別する:
- **更新優先度**: 最初に更新する必要がある vs 後で更新できる
- **依存関係の制約**: 何に依存しているか、何が依存しているか
- **変更スコープ**: 主要（破壊的）、マイナー（互換）、パッチ（修正）

## ステップ 6: ワークフロー可視化の生成

以下を示す Mermaid フローチャートを作成する:
- すべてのフェーズを順序通りに
- 各条件付きフェーズの EXECUTE または SKIP の決定
- 各フェーズ状態に適切なスタイルを適用

**スタイリングルール**（フローチャートの後に追加する）:
```
style WD fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
style CG fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
style BT fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
style US fill:#BDBDBD,stroke:#424242,stroke-width:2px,stroke-dasharray: 5 5,color:#000
style Start fill:#CE93D8,stroke:#6A1B9A,stroke-width:3px,color:#000
style End fill:#CE93D8,stroke:#6A1B9A,stroke-width:3px,color:#000

linkStyle default stroke:#333,stroke-width:2px
```

**スタイルガイドライン**:
- 完了済み/常時実行: `fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff`（マテリアルグリーン、白テキスト）
- 条件付き EXECUTE: `fill:#FFA726,stroke:#E65100,stroke-width:3px,stroke-dasharray: 5 5,color:#000`（マテリアルオレンジ、黒テキスト）
- 条件付き SKIP: `fill:#BDBDBD,stroke:#424242,stroke-width:2px,stroke-dasharray: 5 5,color:#000`（マテリアルグレー、黒テキスト）
- 開始/終了: `fill:#CE93D8,stroke:#6A1B9A,stroke-width:3px,color:#000`（マテリアルパープル、黒テキスト）
- フェーズコンテナ: より淡いマテリアルカラーを使用する（INCEPTION: #BBDEFB、CONSTRUCTION: #C8E6C9、OPERATIONS: #FFF59D）

## ステップ 7: 実行計画ドキュメントの作成

`aidlc-docs/inception/plans/execution-plan.md` を作成する:

```markdown
# Execution Plan

## Detailed Analysis Summary

### Transformation Scope (Brownfield Only)
- **Transformation Type**: [Single component/Architectural/Infrastructure]
- **Primary Changes**: [Description]
- **Related Components**: [List]

### Change Impact Assessment
- **User-facing changes**: [Yes/No - Description]
- **Structural changes**: [Yes/No - Description]
- **Data model changes**: [Yes/No - Description]
- **API changes**: [Yes/No - Description]
- **NFR impact**: [Yes/No - Description]

### Component Relationships (Brownfield Only)
[Component dependency graph]

### Risk Assessment
- **Risk Level**: [Low/Medium/High/Critical]
- **Rollback Complexity**: [Easy/Moderate/Difficult]
- **Testing Complexity**: [Simple/Moderate/Complex]

## Workflow Visualization

```mermaid
flowchart TD
    Start(["User Request"])
    
    subgraph INCEPTION["🔵 INCEPTION PHASE"]
        WD["Workspace Detection<br/><b>STATUS</b>"]
        RE["Reverse Engineering<br/><b>STATUS</b>"]
        RA["Requirements Analysis<br/><b>STATUS</b>"]
        US["User Stories<br/><b>STATUS</b>"]
        WP["Workflow Planning<br/><b>STATUS</b>"]
        AD["Application Design<br/><b>STATUS</b>"]
        UG["Units Generation<br/>(Planning + Generation)<br/><b>STATUS</b>"]
    end
    
    subgraph CONSTRUCTION["🟢 CONSTRUCTION PHASE"]
        FD["Functional Design<br/><b>STATUS</b>"]
        NFRA["NFR Requirements<br/><b>STATUS</b>"]
        NFRD["NFR Design<br/><b>STATUS</b>"]
        ID["Infrastructure Design<br/><b>STATUS</b>"]
        CG["Code Generation<br/>(Planning + Generation)<br/><b>EXECUTE</b>"]
        BT["Build and Test<br/><b>EXECUTE</b>"]
    end
    
    subgraph OPERATIONS["🟡 OPERATIONS PHASE"]
        OPS["Operations<br/><b>PLACEHOLDER</b>"]
    end
    
    Start --> WD
    WD --> RA
    RA --> WP
    WP --> CG
    CG --> BT
    BT --> End(["Complete"])
    
    %% Replace STATUS with COMPLETED, SKIP, EXECUTE as appropriate
    %% Apply styling based on status
```

**注**: STATUS プレースホルダーを実際のフェーズステータス（COMPLETED/SKIP/EXECUTE）に置き換え、適切なスタイルを適用する

## Phases to Execute

### 🔵 INCEPTION PHASE
- [x] Workspace Detection (COMPLETED)
- [x] Reverse Engineering (COMPLETED/SKIPPED)
- [x] Requirements Analysis (COMPLETED)
- [x] User Stories (COMPLETED/SKIPPED)
- [x] Execution Plan (IN PROGRESS)
- [ ] Application Design - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]
- [ ] Units Generation - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]
- [ ] NFR Requirements - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]
- [ ] NFR Design - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]
- [ ] Infrastructure Design - [EXECUTE/SKIP]
  - **Rationale**: [Why executing or skipping]
- [ ] Code Generation - EXECUTE (ALWAYS)
  - **Rationale**: Implementation planning and code generation needed
- [ ] Build and Test - EXECUTE (ALWAYS)
  - **Rationale**: Build, test, and verification needed

### 🟡 OPERATIONS PHASE
- [ ] Operations - PLACEHOLDER
  - **Rationale**: Future deployment and monitoring workflows

## Package Change Sequence (Brownfield Only)
[If applicable, list package update sequence with dependencies]

## Estimated Timeline
- **Total Phases**: [Number]
- **Estimated Duration**: [Time estimate]

## Success Criteria
- **Primary Goal**: [Main objective]
- **Key Deliverables**: [List]
- **Quality Gates**: [List]

[IF brownfield]
- **Integration Testing**: All components working together
- **Operational Readiness**: Monitoring, logging, alerting working
```

## ステップ 8: ステート追跡の初期化

`aidlc-docs/aidlc-state.md` を更新する:

```markdown
# AI-DLC State Tracking

## Project Information
- **Project Type**: [Greenfield/Brownfield]
- **Start Date**: [ISO timestamp]
- **Current Stage**: INCEPTION - Workflow Planning

## Execution Plan Summary
- **Total Stages**: [Number]
- **Stages to Execute**: [List]
- **Stages to Skip**: [List with reasons]

## Stage Progress

### 🔵 INCEPTION PHASE
- [x] Workspace Detection
- [x] Reverse Engineering (if applicable)
- [x] Requirements Analysis
- [x] User Stories (if applicable)
- [x] Workflow Planning
- [ ] Application Design - [EXECUTE/SKIP]
- [ ] Units Generation - [EXECUTE/SKIP]

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design - [EXECUTE/SKIP]
- [ ] NFR Requirements - [EXECUTE/SKIP]
- [ ] NFR Design - [EXECUTE/SKIP]
- [ ] Infrastructure Design - [EXECUTE/SKIP]
- [ ] Code Generation - EXECUTE
- [ ] Build and Test - EXECUTE

### 🟡 OPERATIONS PHASE
- [ ] Operations - PLACEHOLDER

## Current Status
- **Lifecycle Phase**: INCEPTION
- **Current Stage**: Workflow Planning Complete
- **Next Stage**: [Next stage to execute]
- **Status**: Ready to proceed
```

## ステップ 9: 計画をユーザーに提示する

```markdown
# 📋 Workflow Planning Complete

I've created a comprehensive execution plan based on:
- Your request: [Summary]
- Existing system: [Summary if brownfield]
- Requirements: [Summary if executed]
- User stories: [Summary if executed]

**Detailed Analysis**:
- Risk level: [Level]
- Impact: [Summary of key impacts]
- Components affected: [List]

**Recommended Execution Plan**:

I recommend executing [X] stages:

🔵 **INCEPTION PHASE:**
1. [Stage name] - *Rationale:* [Why executing]
2. [Stage name] - *Rationale:* [Why executing]
...

🟢 **CONSTRUCTION PHASE:**
3. [Stage name] - *Rationale:* [Why executing]
4. [Stage name] - *Rationale:* [Why executing]
...

I recommend skipping [Y] stages:

🔵 **INCEPTION PHASE:**
1. [Stage name] - *Rationale:* [Why skipping]
2. [Stage name] - *Rationale:* [Why skipping]
...

🟢 **CONSTRUCTION PHASE:**
3. [Stage name] - *Rationale:* [Why skipping]
4. [Stage name] - *Rationale:* [Why skipping]
...

[IF brownfield with multiple packages]
**Recommended Package Update Sequence**:
1. [Package] - [Reason]
2. [Package] - [Reason]
...

**Estimated Timeline**: [Duration]

> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the execution plan at: `aidlc-docs/inception/plans/execution-plan.md`

> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** - Ask for modifications to the execution plan if required
> [IF any stages are skipped:]
> 📝 **Add Skipped Stages** - Choose to include stages currently marked as SKIP
> ✅ **Approve & Continue** - Approve plan and proceed to **[Next Stage Name]**
```

## ステップ 10: ユーザーレスポンスの処理

- **承認された場合**: 実行計画の次のステージに進む
- **変更が要求された場合**: 実行計画を更新して再確認する
- **ユーザーがステージの強制追加/除外を望む場合**: それに応じて計画を更新する

## ステップ 11: インタラクションの記録

`aidlc-docs/audit.md` に記録する:

```markdown
## Workflow Planning - Approval
**Timestamp**: [ISO timestamp]
**AI Prompt**: "Ready to proceed with this plan?"
**User Response**: "[User's COMPLETE RAW response]"
**Status**: [Approved/Changes Requested]
**Context**: Workflow plan created with [X] stages to execute

---
```
