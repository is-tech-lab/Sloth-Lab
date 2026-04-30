# 機能設計 (Functional Design)

## 目的
**ユニットごとの詳細なビジネスロジック設計**

機能設計 (Functional Design) が対象とする内容:
- ユニットのための詳細なビジネスロジックとアルゴリズム
- エンティティと関係を含むドメインモデル
- 詳細なビジネスルール、バリデーションロジック、および制約
- 技術非依存の設計（インフラの懸念事項は含まない）

**注意**: これはインセプション・フェーズ (INCEPTION PHASE) のアプリケーション設計 (Application Design) における高レベルコンポーネント設計を基に構築されます

## 前提条件
- ユニット生成 (Units Generation) が完了していること
- ユニット・オブ・ワーク (unit of work) の成果物が利用可能であること
- アプリケーション設計 (Application Design) を推奨（高レベルコンポーネント構造を提供）
- 実行計画において機能設計 (Functional Design) ステージを実行することが示されていること

## 概要
技術非依存でビジネス機能のみに焦点を当て、ユニットの詳細なビジネスロジックを設計します。

## 実行ステップ

### ステップ1: ユニットのコンテキストを分析する
- `aidlc-docs/inception/application-design/unit-of-work.md` からユニット定義を読み込む
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md` から割り当てられたストーリーを読み込む
- ユニットの責務と境界を理解する

### ステップ2: 機能設計計画を作成する
- 機能設計用のチェックボックス [] 付きの計画を生成する
- ビジネスロジック、ドメインモデル、ビジネスルールに焦点を当てる
- 各ステップにチェックボックス [] を設ける

### ステップ3: 状況に適した質問を生成する
**指示**: ユニット定義と機能設計の成果物を徹底的に分析し、機能設計の品質向上につながる明確化が必要なすべての領域を特定すること。包括的な理解を確保するために、質問を積極的に行うこと。

**重要 (CRITICAL)**: 機能設計の品質に影響しうる曖昧さや不足している詳細が少しでもあれば、質問することをデフォルトとすること。誤った仮定を立てるよりも、多くの質問をする方が望ましい。

- `[Answer]:` タグ形式を使用して質問を埋め込む
- 曖昧さ、不足情報、または明確化が必要な領域について質問する
- ユーザーの入力が機能設計の意思決定を改善するすべての箇所で質問を生成する
- **迷ったら質問する** — 過度な自信は設計品質の低下につながる

**考慮すべき質問カテゴリ** (すべてのカテゴリを評価すること):
- **ビジネスロジックのモデリング** — コアエンティティ、ワークフロー、データ変換、ビジネスプロセスについて質問する
- **ドメインモデル** — ドメインの概念、エンティティの関係、データ構造、ビジネスオブジェクトについて質問する
- **ビジネスルール** — 意思決定ルール、バリデーションロジック、制約、ビジネスポリシーについて質問する
- **データフロー** — データの入出力、変換、および永続化要件について質問する
- **統合ポイント** — 外部システムとのやり取り、API、およびデータ交換について質問する
- **エラーハンドリング** — エラーシナリオ、バリデーション失敗、および例外処理について質問する
- **ビジネスシナリオ** — エッジケース、代替フロー、および複雑なビジネス状況について質問する
- **フロントエンドコンポーネント** (該当する場合) — UIコンポーネント構造、ユーザーインタラクション、状態管理、フォームハンドリングについて質問する

### ステップ4: 計画を保存する
- `aidlc-docs/construction/plans/{unit-name}-functional-design-plan.md` として保存する
- ユーザー入力用の `[Answer]:` タグをすべて含める

### ステップ5: 回答を収集・分析する
- ユーザーがすべての `[Answer]:` タグを記入するまで待機する
- **必須 (MANDATORY)**: 曖昧または不明確な回答がないか、すべての回答を注意深くレビューする
- **重要 (CRITICAL)**: 不明確な回答に対しては必ずフォローアップ質問を追加する — 曖昧さを残したまま進めない
- 「depends」「maybe」「not sure」「mix of」「somewhere between」といった回答に注意する
- 曖昧さが検出された場合は明確化質問ファイルを作成する
- **すべての曖昧さが解消されるまで進めない**

### ステップ6: 機能設計の成果物を生成する
- `aidlc-docs/construction/{unit-name}/functional-design/business-logic-model.md` を作成する
- `aidlc-docs/construction/{unit-name}/functional-design/business-rules.md` を作成する
- `aidlc-docs/construction/{unit-name}/functional-design/domain-entities.md` を作成する
- ユニットにフロントエンド/UIが含まれる場合: `aidlc-docs/construction/{unit-name}/functional-design/frontend-components.md` を作成する
  - コンポーネント階層と構造
  - 各コンポーネントのPropsと状態の定義
  - ユーザーインタラクションフロー
  - フォームバリデーションルール
  - API統合ポイント（各コンポーネントが使用するバックエンドエンドポイント）

### ステップ7: 完了メッセージを提示する
- 完了メッセージを以下の構成で提示する:
     1. **完了アナウンス** (必須): 常に以下で始めること:

```markdown
# 🔧 Functional Design Complete - [unit-name]
```

     2. **AI要約** (任意): 機能設計の構造化された箇条書き要約を提供する
        - 形式: 「Functional design has created [description]:」
        - 主要なビジネスロジックモデルとエンティティを列挙する（箇条書き）
        - 定義されたビジネスルールとバリデーションロジックを列挙する
        - ドメインモデルの構造と関係について言及する
        - ワークフロー指示は含めない（「please review」「let me know」「proceed to next phase」「before we proceed」など）
        - 事実に基づき、コンテンツに焦点を当てること
     3. **フォーマットされたワークフローメッセージ** (必須): 常に以下の正確な形式で終わること:

```markdown
> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the functional design artifacts at: `aidlc-docs/construction/[unit-name]/functional-design/`



> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** - Ask for modifications to the functional design based on your review  
> ✅ **Continue to Next Stage** - Approve functional design and proceed to **[next-stage-name]**

---
```

### ステップ8: 明示的な承認を待つ
- ユーザーが機能設計を明示的に承認するまで進めない
- 承認は明確かつ曖昧さのないものであること
- ユーザーが変更を要求した場合は、設計を更新して承認プロセスを繰り返す

### ステップ9: 承認を記録し、進捗を更新する
- タイムスタンプ付きで承認を `audit.md` に記録する
- タイムスタンプ付きでユーザーの承認回答を記録する
- `aidlc-state.md` において機能設計 (Functional Design) ステージを完了としてマークする
