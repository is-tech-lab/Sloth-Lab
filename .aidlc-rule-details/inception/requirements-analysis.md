# 要件分析 (Requirements Analysis)（適応型）

**担当ロール**: プロダクトオーナー

**適応型フェーズ**: 常時実行 (ALWAYS EXECUTE)。詳細レベルは問題の複雑さに応じて適応する。

**適応深度の説明については [depth-levels.md](../common/depth-levels.md) を参照**

## 前提条件
- ワークスペース検出 (Workspace Detection) が完了していること
- ブラウンフィールド (Brownfield) の場合、リバースエンジニアリング (Reverse Engineering) が完了していること

## 実行ステップ

### ステップ 1: リバースエンジニアリングコンテキストの読み込み（利用可能な場合）

**ブラウンフィールド (Brownfield) プロジェクトの場合**:
- `aidlc-docs/inception/reverse-engineering/architecture.md` を読み込む
- `aidlc-docs/inception/reverse-engineering/component-inventory.md` を読み込む
- `aidlc-docs/inception/reverse-engineering/technology-stack.md` を読み込む
- リクエストを分析する際に、これらを使用して既存システムを理解する

### ステップ 2: ユーザーリクエストの分析（意図分析）

#### 2.1 リクエストの明確さ
- **明確**: 具体的で、明確に定義されており、実行可能
- **曖昧**: 一般的で、あいまいで、明確化が必要
- **不完全**: 重要な情報が欠けている

#### 2.2 リクエストタイプ
- **新機能 (New Feature)**: 新しい機能の追加
- **バグ修正 (Bug Fix)**: 既存の問題の修正
- **リファクタリング (Refactoring)**: コード構造の改善
- **アップグレード (Upgrade)**: 依存関係またはフレームワークの更新
- **マイグレーション (Migration)**: 別の技術への移行
- **機能強化 (Enhancement)**: 既存機能の改善
- **新規プロジェクト (New Project)**: ゼロからの開始

#### 2.3 初期スコープ見積もり
- **単一ファイル (Single File)**: 1つのファイルへの変更
- **単一コンポーネント (Single Component)**: 1つのコンポーネント/パッケージへの変更
- **複数コンポーネント (Multiple Components)**: 複数のコンポーネントにまたがる変更
- **システム全体 (System-wide)**: システム全体に影響する変更
- **クロスシステム (Cross-system)**: 複数のシステムに影響する変更

#### 2.4 初期複雑性見積もり
- **些細 (Trivial)**: シンプルで簡単な変更
- **単純 (Simple)**: 明確な実装パスがある
- **中程度 (Moderate)**: ある程度の複雑性があり、複数の考慮事項がある
- **複雑 (Complex)**: 高い複雑性があり、多くの考慮事項がある

### ステップ 3: 要件深度の決定

**リクエスト分析に基づいて、深度を決定する:**

**最小深度 (Minimal Depth)** — 以下の場合に使用する:
- リクエストが明確でシンプル
- 詳細な要件が不要
- 基本的な理解を文書化するだけでよい

**標準深度 (Standard Depth)** — 以下の場合に使用する:
- リクエストの明確化が必要
- 機能要件と非機能要件が必要
- 通常の複雑性

**包括深度 (Comprehensive Depth)** — 以下の場合に使用する:
- 複数のステークホルダーを持つ複雑なプロジェクト
- 高リスクまたは重要なシステム
- トレーサビリティを持つ詳細な要件が必要

### ステップ 4: 現在の要件の評価

ユーザーが提供したものを分析する:
   - 意図の記述や説明（すでに `audit.md` に記録済み）
   - 既存の要件ドキュメント（言及された場合はワークスペースを検索する）
   - 貼り付けたコンテンツやファイル参照
   - 非マークダウンドキュメントをマークダウン形式に変換する

### ステップ 5: 徹底的な完全性分析

**重要 (CRITICAL)**: 要件の完全性を評価するために包括的な分析を使用する。曖昧さや不足している詳細が少しでもある場合は、デフォルトで質問する。

**必須 (MANDATORY)**: 以下のすべての領域を評価し、不明確な箇所については質問する:
- **機能要件 (Functional Requirements)**: コア機能、ユーザーインタラクション、システム動作
- **非機能要件 (Non-Functional Requirements)**: パフォーマンス、セキュリティ、スケーラビリティ、ユーザビリティ
- **ユーザーシナリオ (User Scenarios)**: ユースケース、ユーザージャーニー、エッジケース、エラーシナリオ
- **ビジネスコンテキスト (Business Context)**: 目標、制約、成功基準、ステークホルダーのニーズ
- **技術コンテキスト (Technical Context)**: 統合ポイント、データ要件、システム境界
- **品質属性 (Quality Attributes)**: 信頼性、保守性、テスト容易性、アクセシビリティ

**疑問がある場合は質問する** — 不完全な要件は質の低い実装につながる。

### ステップ 5.1: 拡張機能のオプトイン確認

**必須 (MANDATORY)**: 読み込まれたすべての `*.opt-in.md` ファイル（ワークフロー開始時に `extensions/` サブディレクトリから読み込まれたもの）を `## Opt-In Prompt` セクションでスキャンする。そのセクションを宣言している各拡張機能について、ステップ 6 で作成する確認質問ファイルにその質問を含める。各オプトイン質問は、ユーザーの会話と同じ言語で提示する。

回答を受け取った後:
1. 各拡張機能の有効化ステータスを `aidlc-docs/aidlc-state.md` の `## Extension Configuration` に記録する:

```markdown
## Extension Configuration
| Extension | Enabled | Decided At |
|---|---|---|
| [Extension Name] | [Yes/No] | Requirements Analysis |
```

2. **遅延ルール読み込み (Deferred Rule Loading)**: ユーザーがオプトインした各拡張機能について、今すぐ完全なルールファイルを読み込む。ルールファイルは命名規則により導出される: オプトインファイル名から `.opt-in.md` を削除し `.md` を追加する（例: `security-baseline.opt-in.md` → `security-baseline.md`）。ユーザーがオプトアウトした拡張機能については、完全なルールファイルを読み込まない。

### ステップ 6: 確認質問の生成（積極的アプローチ）
   - 要件が例外的に明確かつ完全でない限り、**常に** `aidlc-docs/inception/requirements/requirement-verification-questions.md` を作成する
   - 不足している、不明確な、または曖昧な領域について質問する
   - 機能要件、非機能要件、ユーザーシナリオ、ビジネスコンテキストに焦点を当てる
   - ユーザーが質問ドキュメント内のすべての `[Answer]:` タグを直接入力するよう依頼する
   - 回答の選択肢を複数提示する場合:
     - オプションを A, B, C, D などでラベル付けする
     - オプションは相互に排他的で重複しないようにする
     - 常にカスタム回答のオプションを含める: "X) その他（`[Answer]:` タグの後に記述してください）"
   - ドキュメント内でユーザーの回答を待つ
   - **必須 (MANDATORY)**: すべての回答の曖昧さを分析し、必要に応じてフォローアップ質問を作成する
   - **必須 (MANDATORY)**: すべての曖昧さが解消されるか、ユーザーが明示的に進めるよう要求するまで質問を続ける

### ⛔ ゲート: ユーザーの回答を待つ
`requirement-verification-questions.md` のすべての質問が回答され検証されるまで、ステップ 7 に進んではならない。
質問ファイルをユーザーに提示して停止する。

### ステップ 7: 要件ドキュメントの生成
   - **前提条件**: ステップ 6 のゲートを通過していること — すべての回答を受け取り分析済みであること
   - `aidlc-docs/inception/requirements/requirements.md` を作成する
   - 先頭に意図分析サマリーを含める:
     - ユーザーリクエスト
     - リクエストタイプ
     - スコープ見積もり
     - 複雑性見積もり
   - 機能要件と非機能要件の両方を含める
   - 確認質問へのユーザーの回答を組み込む
   - 主要な要件の簡潔なサマリーを提供する

### ステップ 8: ステート追跡の更新

`aidlc-docs/aidlc-state.md` を更新する:

```markdown
## Stage Progress
### 🔵 INCEPTION PHASE
- [x] Workspace Detection
- [x] Reverse Engineering (if applicable)
- [x] Requirements Analysis
```

### ステップ 9: 記録と次フェーズへの進行
   - 承認プロンプトをタイムスタンプとともに `aidlc-docs/audit.md` に記録する
   - 以下の構造で完了メッセージを提示する:
     1. **完了アナウンス** (必須): 常にこれで始める:

```markdown
# 🔍 Requirements Analysis Complete
```

     2. **AI サマリー** (任意): 要件の構造化された箇条書きサマリーを提供する
        - 形式: "Requirements analysis has identified [project type/complexity]:"
        - 主要な機能要件をリストアップする（箇条書き）
        - 主要な非機能要件をリストアップする（箇条書き）
        - 関連する場合はアーキテクチャ上の考慮事項や技術的な決定に言及する
        - ワークフローの指示は含めない（"please review"、"let me know"、"proceed to next phase"、"before we proceed" など）
        - 事実に基づいた内容重視の記述にとどめる
     3. **フォーマット済みワークフローメッセージ** (必須): 常にこの正確な形式で終了する:

```markdown
> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the requirements document at: `aidlc-docs/inception/requirements/requirements.md`



> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** -  Ask for modifications to the requirements if required based on your review 
> [IF User Stories will be skipped, add this option:]
> 📝 **Add User Stories** - Choose to Include **User Stories** stage (currently skipped based on project simplicity)  
> ✅ **Approve & Continue** - Approve requirements and proceed to **[User Stories/Workflow Planning]**

---
```

**注**: "Add User Stories" オプションは、ユーザーストーリー (User Stories) ステージがスキップされる場合にのみ含める。`[User Stories/Workflow Planning]` は実際の次ステージ名に置き換える。

   - ユーザーの明示的な承認を待ってから次に進む
   - 承認レスポンスをタイムスタンプとともに記録する
   - `aidlc-state.md` で要件分析 (Requirements Analysis) ステージの完了を更新する
