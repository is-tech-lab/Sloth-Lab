# セッション継続テンプレート

## ウェルカムバックプロンプトテンプレート
ユーザーが既存の AI-DLC プロジェクトの作業を再開するために戻ってきた際は、以下のプロンプトを表示します:

```markdown
**おかえりなさい！進行中の AI-DLC プロジェクトが見つかりました。**

aidlc-state.md に基づく現在のステータス:
- **プロジェクト**: [project-name]
- **現在のフェーズ**: [INCEPTION/CONSTRUCTION/OPERATIONS]
- **現在のステージ**: [Stage Name]
- **最後に完了したステップ**: [Last completed step]
- **次のステップ**: [Next step to work on]

**本日はどこから作業を続けますか？**

A) 前回の続きから再開する ([Next step description])
B) 前のステージを確認する ([Show available stages])

[Answer]: 
```

## 必須 (MANDATORY): セッション継続の手順
1. **既存プロジェクトを検出した際は常に `aidlc-state.md` を最初に読む**
2. **ワークフローファイルから現在のステータスを解析**してプロンプトに反映する
3. **必須 (MANDATORY): 前のステージの成果物を読み込む** - いずれかのステージを再開する前に、前のステージから関連するすべての成果物を自動的に読み込む:
   - **リバースエンジニアリング (Reverse Engineering)**: `architecture.md`、`code-structure.md`、`api-documentation.md` を読み込む
   - **要件分析 (Requirements Analysis)**: `requirements.md`、`requirement-verification-questions.md` を読み込む
   - **ユーザーストーリー (User Stories)**: `stories.md`、`personas.md`、`story-generation-plan.md` を読み込む
   - **アプリケーション設計 (Application Design)**: アプリケーション設計の成果物 (`components.md`、`component-methods.md`、`services.md`) を読み込む
   - **設計 (ユニット)**: `unit-of-work.md`、`unit-of-work-dependency.md`、`unit-of-work-story-map.md` を読み込む
   - **ユニットごとの設計**: `functional-design.md`、`nfr-requirements.md`、`nfr-design.md`、`infrastructure-design.md` を読み込む
   - **コードステージ**: すべてのコードファイル、プラン、および前のすべての成果物を読み込む
4. **ステージ別のスマートなコンテキスト読み込み**:
   - **初期ステージ (ワークスペース検出 (Workspace Detection)、リバースエンジニアリング (Reverse Engineering))**: ワークスペース分析を読み込む
   - **要件分析 (Requirements Analysis) / ユーザーストーリー (User Stories)**: リバースエンジニアリング (Reverse Engineering) + 要件の成果物を読み込む
   - **設計ステージ**: 要件 + ユーザーストーリー + アーキテクチャ + 設計の成果物を読み込む
   - **コードステージ**: すべての成果物 + 既存のコードファイルを読み込む
5. **アーキテクチャの選択と現在のフェーズに基づいて選択肢を調整する**
6. **汎用的な説明ではなく、具体的な次のステップを表示する**
7. **継続プロンプトを `audit.md` にタイムスタンプ付きで記録する**
8. **コンテキストサマリー**: 成果物を読み込んだ後、ユーザーへの認識向上のために読み込んだ内容の簡単なサマリーを提供する
9. **質問の提示**: 確認や利用者フィードバックの質問は**常に** .md ファイルに配置すること。複数選択形式の質問をチャットセッション内にインラインで配置してはならない (禁止 (NEVER))。

## エラー処理
セッション再開中に成果物が欠損または破損している場合は、復旧手順について [error-handling.md](error-handling.md) を参照してください。
