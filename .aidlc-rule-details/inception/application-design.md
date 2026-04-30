# アプリケーション設計 (Application Design) - 詳細ステップ

## 目的
**高レベルなコンポーネント識別とサービスレイヤー設計**

アプリケーション設計 (Application Design) の焦点:
- 主要な機能コンポーネントとその責務の識別
- コンポーネントインターフェースの定義（詳細なビジネスロジックは含まない）
- オーケストレーションのためのサービスレイヤーの設計
- コンポーネントの依存関係とコミュニケーションパターンの確立

**注**: 詳細なビジネスロジック設計は、後の機能設計 (Functional Design)（ユニットごと、コンストラクション・フェーズ (CONSTRUCTION PHASE)）で行われる

## 前提条件
- ワークスペース検出 (Workspace Detection) が完了していること
- 要件分析 (Requirements Analysis) を推奨する（機能的なコンテキストを提供する）
- ユーザーストーリー (User Stories) を推奨する（ユーザーストーリーが設計の意思決定をガイドする）
- 実行計画でアプリケーション設計 (Application Design) ステージを実行すると示されていること

## ステップバイステップの実行

### 1. コンテキストの分析
- `aidlc-docs/inception/requirements/requirements.md` と `aidlc-docs/inception/user-stories/stories.md` を読み込む
- 主要なビジネス能力と機能領域を特定する
- 設計スコープと複雑性を決定する

### 2. アプリケーション設計計画の作成
- アプリケーション設計のためのチェックボックス `[]` 付きの計画を生成する
- コンポーネント、責務、メソッド、ビジネスルール、サービスに焦点を当てる
- 各ステップとサブステップにチェックボックス `[]` を設ける

### 3. 必須設計成果物を計画に含める
- 以下の必須成果物を**常に**設計計画に含める:
  - [ ] コンポーネント定義と高レベルの責務を含む components.md を生成する
  - [ ] メソッドシグネチャを含む component-methods.md を生成する（ビジネスルールの詳細は後で機能設計 (Functional Design) で行う）
  - [ ] サービス定義とオーケストレーションパターンを含む services.md を生成する
  - [ ] 依存関係とコミュニケーションパターンを含む component-dependency.md を生成する
  - [ ] 設計の完全性と一貫性を検証する

### 4. コンテキストに適した質問の生成
**指令**: 要件とストーリーを分析し、このアプリケーション設計に関連する質問を生成する。以下のカテゴリーをガイダンスとして使用する。各カテゴリーを評価し、適用可能かどうか迷った場合は、スキップするのではなく質問する — 過信は良くない結果につながる（overconfidence-prevention.md を参照）。

- `[Answer]:` タグ形式を使用して質問を埋め込む
- 曖昧さ、不足している情報、または明確化が必要な領域について質問する
- ユーザーの入力によって設計の意思決定が改善される箇所に質問を生成する
- **迷った場合は質問する** — 過信は質の低い設計につながる

**評価する質問カテゴリー**（すべてのカテゴリーを検討する）:
- **コンポーネント識別 (Component Identification)** — コンポーネントの境界、整理、グループ化戦略について質問する
- **コンポーネントメソッド (Component Methods)** — メソッドシグネチャ、入出力の期待値、インターフェースコントラクトについて質問する（詳細なビジネスルールは後で行う）
- **サービスレイヤー設計 (Service Layer Design)** — サービスのオーケストレーション、境界、調整パターンについて質問する
- **コンポーネント依存関係 (Component Dependencies)** — コミュニケーションパターン、依存関係の管理、結合の懸念について質問する
- **設計パターン (Design Patterns)** — アーキテクチャスタイルの好み、パターンの選択、設計の制約について質問する

### 5. アプリケーション設計計画の保存
- `aidlc-docs/inception/plans/application-design-plan.md` として保存する
- ユーザー入力用のすべての `[Answer]:` タグを含める
- 計画がすべての設計側面をカバーしていることを確認する

### 6. ユーザー入力の要求
- 計画ドキュメント内の `[Answer]:` タグを直接入力するようユーザーに依頼する
- 設計上の意思決定の重要性を強調する
- `[Answer]:` タグの完成方法について明確な指示を提供する

### 7. 回答の収集
- ドキュメント内の `[Answer]:` タグを使用してすべての質問に回答するのをユーザーが完了するのを待つ
- すべての `[Answer]:` タグが完了するまで進まない
- `[Answer]:` タグが空白のまま残っていないことを確認するためにドキュメントをレビューする

### 8. 回答を分析する（必須 (MANDATORY)）
進める前に、すべてのユーザー回答を以下について注意深くレビューしなければならない:
- **曖昧または不明確な回答**: "〜の混合"、"〜の間のどこか"、"よくわからない"、"次第"
- **未定義の基準または用語**: 明確な定義のない概念への参照
- **矛盾した回答**: 互いに矛盾する回答
- **不足している設計詳細**: 具体的な指針がない回答
- **選択肢を組み合わせた回答**: 明確な決定ルールなしに異なるアプローチを混合した回答

### 9. 必須フォローアップ質問
ステップ 8 の分析で曖昧な回答が見つかった場合は、以下を必ず行う:
- `[Answer]:` タグを使用して計画ドキュメントに具体的なフォローアップ質問を追加する
- すべての曖昧さが解消されるまで承認に進まない
- 必要なフォローアップの例:
  - "「A と B の混合」と述べていましたが、A と B のどちらを使うかを決める具体的な基準は何ですか？"
  - "「A と B の間のどこか」と述べていましたが、正確な中間アプローチを定義できますか？"
  - "「よくわからない」と述べていましたが、決定に役立つ追加情報は何ですか？"
  - "「複雑さ次第」と述べていましたが、複雑さのレベルをどのように定義しますか？"

### 10. アプリケーション設計成果物の生成
- 承認された計画を実行して設計成果物を生成する
- `aidlc-docs/inception/application-design/components.md` を作成する（内容）:
  - コンポーネント名と目的
  - コンポーネントの責務
  - コンポーネントインターフェース
- `aidlc-docs/inception/application-design/component-methods.md` を作成する（内容）:
  - 各コンポーネントのメソッドシグネチャ
  - 各メソッドの高レベルな目的
  - 入出力の型
  - 注: 詳細なビジネスルールは機能設計 (Functional Design)（ユニットごと、コンストラクション・フェーズ (CONSTRUCTION PHASE)）で定義される
- `aidlc-docs/inception/application-design/services.md` を作成する（内容）:
  - サービス定義
  - サービスの責務
  - サービスのインタラクションとオーケストレーション
- `aidlc-docs/inception/application-design/component-dependency.md` を作成する（内容）:
  - 関係性を示す依存関係マトリクス
  - コンポーネント間のコミュニケーションパターン
  - データフロー図
- 上記で作成した複数の設計ドキュメントを1つのドキュメントに統合した `aidlc-docs/inception/application-design/application-design.md` を作成する

### 11. 承認の記録
- 承認プロンプトをタイムスタンプとともに `aidlc-docs/audit.md` に記録する
- 完全な承認プロンプトテキストを含める
- ISO 8601 タイムスタンプ形式を使用する

### 12. 完了メッセージの提示

```markdown
# 🏗️ Application Design Complete

[AI-generated summary of application design artifacts created in bullet points]

> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the application design artifacts at: `aidlc-docs/inception/application-design/`

> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** - Ask for modifications to the application design if required
> [IF Units Generation is skipped:]
> 📝 **Add Units Generation** - Choose to include **Units Generation** stage (currently skipped)
> ✅ **Approve & Continue** - Approve design and proceed to **[Units Generation/CONSTRUCTION PHASE]**
```

### 13. 明示的な承認を待つ
- ユーザーがアプリケーション設計を明示的に承認するまで進まない
- 承認は明確で曖昧でないものでなければならない
- ユーザーが変更を要求した場合は、設計を更新して承認プロセスを繰り返す

### 14. 承認レスポンスの記録
- ユーザーの承認レスポンスをタイムスタンプとともに `aidlc-docs/audit.md` に記録する
- 正確なユーザー回答テキストを含める
- 承認ステータスを明確にマークする

### 15. 進捗の更新
- `aidlc-docs/aidlc-state.md` でアプリケーション設計 (Application Design) ステージの完了をマークする
- "Current Status" セクションを更新する
- 次のステージへの移行を準備する
