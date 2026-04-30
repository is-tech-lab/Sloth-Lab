# ワークスペース検出 (Workspace Detection)

**目的**: ワークスペースの状態を確認し、既存の AI-DLC プロジェクトをチェックする

## ステップ 1: 既存の AI-DLC プロジェクトの確認

`aidlc-docs/aidlc-state.md` が存在するか確認する:
- **存在する場合**: 最後のフェーズから再開する（以前のフェーズのコンテキストを読み込む）
- **存在しない場合**: 新規プロジェクトの評価を続行する

## ステップ 2: 既存コードのワークスペーススキャン

**ワークスペースに既存コードがあるかどうかを判定する:**
- ワークスペース内のソースコードファイル (.java, .py, .js, .ts, .jsx, .tsx, .kt, .kts, .scala, .groovy, .go, .rs, .rb, .php, .c, .h, .cpp, .hpp, .cc, .cs, .fs など) をスキャンする
- ビルドファイル (pom.xml, package.json, build.gradle など) を確認する
- プロジェクト構造の指標を探す
- ワークスペースのルートディレクトリを特定する（`aidlc-docs/` は除く）

**調査結果を記録する:**
```markdown
## Workspace State
- **Existing Code**: [Yes/No]
- **Programming Languages**: [List if found]
- **Build System**: [Maven/Gradle/npm/etc. if found]
- **Project Structure**: [Monolith/Microservices/Library/Empty]
- **Workspace Root**: [Absolute path]
```

## ステップ 3: 次フェーズの決定

**ワークスペースが空の場合（既存コードなし）**:
- フラグを設定する: `brownfield = false`
- 次フェーズ: 要件分析 (Requirements Analysis)

**ワークスペースに既存コードがある場合**:
- フラグを設定する: `brownfield = true`
- `aidlc-docs/inception/reverse-engineering/` に既存のリバースエンジニアリング成果物があるか確認する
- **リバースエンジニアリング成果物が存在する場合**:
    - 成果物が古くなっていないか確認する（成果物のタイムスタンプとコードベースの最終重大更新日時を比較する）
    - **成果物が最新の場合**: それらを読み込み、要件分析 (Requirements Analysis) へスキップする
    - **成果物が古い場合**: 次フェーズはリバースエンジニアリング (Reverse Engineering)（成果物を更新するために再実行）
    - **ユーザーが明示的に再実行を要求した場合**: 古さに関わらず次フェーズはリバースエンジニアリング (Reverse Engineering)
- **リバースエンジニアリング成果物がない場合**: 次フェーズはリバースエンジニアリング (Reverse Engineering)

## ステップ 4: 初期ステートファイルの作成

`aidlc-docs/aidlc-state.md` を作成する:

```markdown
# AI-DLC State Tracking

## Project Information
- **Project Type**: [Greenfield/Brownfield]
- **Start Date**: [ISO timestamp]
- **Current Stage**: INCEPTION - Workspace Detection

## Workspace State
- **Existing Code**: [Yes/No]
- **Reverse Engineering Needed**: [Yes/No]
- **Workspace Root**: [Absolute path]

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Stage Progress
[Will be populated as workflow progresses]
```

## ステップ 5: 完了メッセージの表示

**ブラウンフィールド (Brownfield) プロジェクトの場合:**
```markdown
# 🔍 Workspace Detection Complete

Workspace analysis findings:
• **Project Type**: Brownfield project
• [AI-generated summary of workspace findings in bullet points]
• **Next Step**: Proceeding to **Reverse Engineering** to analyze existing codebase...
```

**グリーンフィールド (Greenfield) プロジェクトの場合:**
```markdown
# 🔍 Workspace Detection Complete

Workspace analysis findings:
• **Project Type**: Greenfield project
• **Next Step**: Proceeding to **Requirements Analysis**...
```

## ステップ 6: 自動的に次フェーズへ進む

- **ユーザーの承認は不要** — これは情報提供のみを目的とする
- 次フェーズへ自動的に進む:
  - **ブラウンフィールド (Brownfield)**: リバースエンジニアリング (Reverse Engineering)（既存の成果物がない場合）または要件分析 (Requirements Analysis)（成果物が存在する場合）
  - **グリーンフィールド (Greenfield)**: 要件分析 (Requirements Analysis)
