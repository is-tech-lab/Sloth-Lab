# インフラ設計 (Infrastructure Design)

## 前提条件
- ユニットの機能設計 (Functional Design) が完了していること
- NFR設計 (NFR Design) を推奨（マッピングする論理コンポーネントを提供）
- 実行計画においてインフラ設計 (Infrastructure Design) ステージを実行することが示されていること

## 概要
論理ソフトウェアコンポーネントをデプロイ環境の実際のインフラ選択肢にマッピングします。

## 実行ステップ

### ステップ1: 設計成果物を分析する
- `aidlc-docs/construction/{unit-name}/functional-design/` から機能設計を読み込む
- `aidlc-docs/construction/{unit-name}/nfr-design/` からNFR設計を読み込む（存在する場合）
- インフラが必要な論理コンポーネントを特定する

### ステップ2: インフラ設計計画を作成する
- インフラ設計用のチェックボックス [] 付きの計画を生成する
- 実際のサービス（AWS、Azure、GCP、オンプレミス）へのマッピングに焦点を当てる
- 各ステップにチェックボックス [] を設ける

### ステップ3: 状況に適した質問を生成する
**指示**: 機能設計およびNFR設計を徹底的に分析し、インフラの意思決定を改善する明確化が必要なすべての領域を特定すること。包括的なインフラカバレッジを確保するために、質問を積極的に行うこと。

**重要 (CRITICAL)**: インフラ品質に影響しうる曖昧さや不足している詳細が少しでもあれば、質問することをデフォルトとすること。誤ったインフラの仮定を立てるよりも、多くの質問をする方が望ましい。

**必須 (MANDATORY)**: 以下のすべてのカテゴリを評価し、各カテゴリについて的を絞った質問を行うこと。機能設計およびNFR設計の成果物からの根拠に基づいて各カテゴリの適用可能性を判断すること — 明示的な正当理由なくカテゴリをスキップしないこと:

- `[Answer]:` タグ形式を使用して質問を埋め込む
- 曖昧さ、不足情報、または明確化が必要な領域について質問する
- ユーザーの入力がインフラの意思決定を改善するすべての箇所で質問を生成する
- **迷ったら質問する** — 過度な自信はインフラ選択の品質低下につながる

**評価すべき質問カテゴリ** (すべてのカテゴリを考慮すること):
- **デプロイ環境** — クラウドプロバイダーの好み、環境設定、デプロイ対象について質問する
- **コンピューティングインフラ** — コンピューティングサービスの選択、サイジング、スケーリング要件について質問する
- **ストレージインフラ** — データベースの選定、ストレージパターン、データライフサイクルのニーズについて質問する
- **メッセージングインフラ** — メッセージング/キューイングサービス、イベント駆動パターン、非同期処理について質問する
- **ネットワークインフラ** — ロードバランシング、API ゲートウェイのアプローチ、ネットワークトポロジーについて質問する
- **モニタリングインフラ** — オブザーバビリティツール、アラート戦略、ロギング要件について質問する
- **共有インフラ** — インフラ共有戦略、マルチテナンシー、リソース分離について質問する

### ステップ4: 計画を保存する
- `aidlc-docs/construction/plans/{unit-name}-infrastructure-design-plan.md` として保存する
- ユーザー入力用の `[Answer]:` タグをすべて含める

### ステップ5: 回答を収集・分析する
- ユーザーがすべての `[Answer]:` タグを記入するまで待機する
- 曖昧または不明確な回答がないかレビューする
- 必要に応じてフォローアップ質問を追加する

### ステップ6: インフラ設計の成果物を生成する
- `aidlc-docs/construction/{unit-name}/infrastructure-design/infrastructure-design.md` を作成する
- `aidlc-docs/construction/{unit-name}/infrastructure-design/deployment-architecture.md` を作成する
- 共有インフラがある場合: `aidlc-docs/construction/shared-infrastructure.md` を作成する

### ステップ7: 完了メッセージを提示する
- 完了メッセージを以下の構成で提示する:
     1. **完了アナウンス** (必須): 常に以下で始めること:

```markdown
# 🏢 Infrastructure Design Complete - [unit-name]
```

     2. **AI要約** (任意): インフラ設計の構造化された箇条書き要約を提供する
        - 形式: 「Infrastructure design has mapped [description]:」
        - 主要なインフラサービスとコンポーネントを列挙する（箇条書き）
        - デプロイアーキテクチャの決定とその根拠を列挙する
        - クラウドプロバイダーの選択とサービスマッピングについて言及する
        - ワークフロー指示は含めない（「please review」「let me know」「proceed to next phase」「before we proceed」など）
        - 事実に基づき、コンテンツに焦点を当てること
     3. **フォーマットされたワークフローメッセージ** (必須): 常に以下の正確な形式で終わること:

```markdown
> **📋 <u>**REVIEW REQUIRED:**</u>**  
> Please examine the infrastructure design at: `aidlc-docs/construction/[unit-name]/infrastructure-design/`



> **🚀 <u>**WHAT'S NEXT?**</u>**
>
> **You may:**
>
> 🔧 **Request Changes** - Ask for modifications to the infrastructure design based on your review  
> ✅ **Continue to Next Stage** - Approve infrastructure design and proceed to **Code Generation**

---
```

### ステップ8: 明示的な承認を待つ
- ユーザーがインフラ設計を明示的に承認するまで進めない
- 承認は明確かつ曖昧さのないものであること
- ユーザーが変更を要求した場合は、設計を更新して承認プロセスを繰り返す

### ステップ9: 承認を記録し、進捗を更新する
- タイムスタンプ付きで承認を `audit.md` に記録する
- タイムスタンプ付きでユーザーの承認回答を記録する
- `aidlc-state.md` においてインフラ設計 (Infrastructure Design) ステージを完了としてマークする
