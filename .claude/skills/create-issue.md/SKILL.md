---
name: create-issue
description: GitHub Issueを自動作成し、Projects連携を行うスキル。段階的な質問で情報収集、重複チェック、テンプレート適用、ラベル自動判定に対応。
---

# GitHub Issue作成スキル

## ワークフロー概要

```
1. 情報収集 → 2. 重複チェック → 3. テンプレート適用 → 4. Issue作成 → 5. Projects連携
```

---

## Step 1: 情報収集

ユーザーから以下の情報を収集します。

### 必須項目

| 項目 | 説明 | 例 |
|------|------|-----|
| 種別 | Bug/Feature/Refactoring/Documentation/Other | Bug |
| タイトル | `[種別] 具体的な内容` 形式 | [Bug] アイテム削除時に親ディレクトリが更新されない |
| 説明 | 詳細な説明（種別により質問内容が変わる） | 現象、影響範囲、再現手順 |

### 任意項目

| 項目 | デフォルト値 |
|------|-------------|
| 関連ファイル | なし（ラベル判定に使用） |
| 受け入れ条件 | デフォルト値を適用 |
| 優先度 | Medium |
| 親Issue | なし |
| マイルストーン | なし |
| アサイン | @me |

**ヒント**: ユーザーが一度に情報を提供した場合は段階的質問をスキップ可能

---

## Step 2: 重複チェック

Issue作成前に既存Issueを検索し、重複を避けます。

**詳細**: [references/duplicate-detection.md](references/duplicate-detection.md)

### 判定基準

| 判定 | 動作 |
|------|------|
| 完全一致 | 作成中止、既存Issue URLを返す |
| 高い類似性（80%以上） | ユーザーに確認 |
| 低い類似性 | 情報提供のみ、作成続行 |

---

## Step 3: テンプレート適用・ラベル判定

### テンプレート選択

| 種別 | テンプレート |
|------|-------------|
| Bug | `.github/ISSUE_TEMPLATE/bug-template.md` |
| Feature | `.github/ISSUE_TEMPLATE/fixture-template.md` |
| その他 | カスタムフォーマット |

**詳細**: [references/template-processing.md](references/template-processing.md)

### ラベル自動判定

**詳細**: [references/issue-creation-rules.md](references/issue-creation-rules.md)

---

## Step 4: Issue作成

```bash
gh issue create \
  --repo "SPinnoDeveloper/spinno-next" \
  --title "$title" \
  --body "$body" \
  --label "$labels" \
  --assignee "${assignee:-@me}" \
  ${milestone:+--milestone "$milestone"}
```

---

## Step 5: Projects連携（オプション）

GitHub Projects v2と連携し、親子関係や優先度を設定します。

**詳細**: [references/projects-integration.md](references/projects-integration.md)

---

## クイックリファレンス

### ラベル自動判定表

| ファイルパスパターン | コンポーネントラベル |
|-------------------|-------------------|
| `*item*` | `component:items` |
| `*directory*` | `component:directories` |
| `*user*` | `component:users` |
| `*banner*` | `component:banners` |
| `*event*` | `component:events` |
| `src/presenter/ddd/*`, `api-spec/*` | `component:api` |
| `db/public/*` | `component:database` |
| `src/infra/repository/*` | `component:repository` |
| `src/infra/storage/*` | `component:storage` |
| `src/oauth/*`, `*auth*` | `component:auth` |
| その他 | `component:other` |

### タイプラベル

| 種別 | ラベル |
|------|--------|
| Bug | `type:bug` |
| Feature | `type:feature` |
| Refactoring | `type:refactoring` |
| Documentation | `type:documentation` |
| Other | `type:other` |

### 優先度ラベル

| 優先度 | ラベル |
|--------|--------|
| High | `priority:high` |
| Medium | `priority:medium` |
| Low | `priority:low` |

### タイトル形式

```
[種別] 具体的な内容
```

**例**:
- `[Bug] アイテム削除時に親ディレクトリが更新されない`
- `[Feature] アイテム一括削除機能の追加`
- `[Refactoring] Item entityのビジネスロジック整理`

---

## 結果報告

Issue作成後、以下の形式で報告します。

```
✅ GitHub Issueを作成しました

Issue URL: {issue_url}
```

---

## 参照ファイル

| ファイル | 内容 |
|---------|------|
| [references/duplicate-detection.md](references/duplicate-detection.md) | 検索・重複チェックロジック |
| [references/template-processing.md](references/template-processing.md) | テンプレート処理 |
| [references/projects-integration.md](references/projects-integration.md) | GitHub Projects連携 |
| [references/issue-creation-rules.md](references/issue-creation-rules.md) | ラベル体系・命名規則 |

---

## 注意事項

- Issue作成のみを行い、コード修正は行わない
- Issue作成後の編集やコメント追加は行わない
- 必須情報が不足している場合はユーザーに質問する
- すべての説明とコメントは日本語で記述
- エラー発生時は具体的なエラー内容と対処法を提示