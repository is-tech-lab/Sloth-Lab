# Execution Plan — Sloth Feed PoC

## 詳細分析サマリー

### 変更影響評価

| 影響領域 | 有無 | 内容 |
|---------|------|------|
| ユーザー向けの変更 | Yes | 全機能がユーザー直接操作（投稿・フィード・AIコメント） |
| 構造的な変更 | Yes | 新規システム。Next.js + DynamoDB + Claude API の統合設計が必要 |
| データモデルの変更 | Yes | User・Post の新規設計 |
| API の変更 | Yes | auth / posts / feed / ai-comment の新規エンドポイント設計 |
| NFR への影響 | Yes | Claude API レスポンスタイム・DynamoDB 設計・JWT 認証 |

### リスク評価

| 項目 | 評価 |
|------|------|
| **リスクレベル** | Medium |
| **ロールバック複雑性** | Moderate（AWS インフラと Claude API の依存関係あり） |
| **テスト複雑性** | Moderate（AI フィルタリングの精度検証が必要） |
| **不確実性** | Claude API のフィルタリング精度（PoC レベルで許容） |

---

## ワークフロー可視化

```
[INCEPTION PHASE]
  WD  : Workspace Detection      → COMPLETED
  RE  : Reverse Engineering      → SKIPPED (Greenfield)
  RA  : Requirements Analysis    → COMPLETED
  US  : User Stories             → COMPLETED
  WP  : Workflow Planning        → COMPLETED (now)
  AD  : Application Design       → EXECUTE
  UG  : Units Generation         → EXECUTE

[CONSTRUCTION PHASE] (ユニットごとのループ)
  FD  : Functional Design        → EXECUTE
  NFRA: NFR Requirements         → EXECUTE
  NFRD: NFR Design               → EXECUTE
  ID  : Infrastructure Design    → EXECUTE
  CG  : Code Generation          → EXECUTE (ALWAYS)
  BT  : Build and Test           → EXECUTE (ALWAYS)

[OPERATIONS PHASE]
  OPS : Operations               → PLACEHOLDER
```

---

## 実行するフェーズ

### 🔵 INCEPTION PHASE
- [x] Workspace Detection — COMPLETED
- [x] Reverse Engineering — SKIPPED (Greenfield)
- [x] Requirements Analysis — COMPLETED
- [x] User Stories — COMPLETED
- [x] Workflow Planning — COMPLETED
- [ ] Application Design — **EXECUTE**
  - **理由**: 新規コンポーネント（Auth / Post / Feed / AIComment）が必要。サービスレイヤー設計と依存関係の明確化が必要
- [ ] Units Generation — **EXECUTE**
  - **理由**: 新規データモデル・APIエンドポイント・複数ユニットへの分解が必要。AIフィルタリングとAIコメントは独立したユニットとして設計する

### 🟢 CONSTRUCTION PHASE（ユニットごとのループ）
- [ ] Functional Design — **EXECUTE**
  - **理由**: 新規データモデル（User・Post）・AIフィルタリングロジック・AIコメント生成ロジックの詳細設計が必要
- [ ] NFR Requirements — **EXECUTE**
  - **理由**: Claude API レスポンスタイム・DynamoDB 設計パターン・JWT 認証・APIコスト管理の要件定義が必要
- [ ] NFR Design — **EXECUTE**
  - **理由**: NFR Requirements 実行のため。エラーハンドリング・ロギング設計を含む
- [ ] Infrastructure Design — **EXECUTE**
  - **理由**: AWS DynamoDB テーブル設計・Next.js on AWS デプロイ構成・Claude API 統合設計が必要
- [ ] Code Generation — **EXECUTE** (ALWAYS)
- [ ] Build and Test — **EXECUTE** (ALWAYS)

### 🟡 OPERATIONS PHASE
- [ ] Operations — PLACEHOLDER（将来のデプロイ・監視ワークフロー）

---

## ユニット分解（Units Generation で詳細化）

| Unit | 名称 | 内容 |
|------|------|------|
| Unit 1 | 認証（Auth） | メール+パスワード登録・ログイン・JWT発行・DynamoDB User テーブル |
| Unit 2 | 投稿 + AIフィルタリング（Post） | テキスト投稿 → Claude API フィルタリング → 除外フィードバック生成 |
| Unit 3 | AIコメント生成（AIComment） | フィルタリング通過後の称賛コメント生成（偉人・論文引用） |
| Unit 4 | フィード・投稿履歴（Feed） | タイムライン一覧・自分の投稿一覧 |

**実行順序**: Unit 1（Auth）→ Unit 2（Post + Filtering）→ Unit 3（AIComment）→ Unit 4（Feed）

---

## 成功基準

- **Primary Goal**: 「仕事じゃないけど」を投稿 → AIフィルタリング → AIコメント → タイムライン表示 の一連フローが動作する
- **Key Deliverables**: 認証・投稿・AIフィルタリング・AIコメント・タイムライン・過去投稿一覧
- **Quality Gates**:
  - Claude API が仕事成果・旅行投稿を正しく除外する
  - Claude API が投稿内容に応じた称賛コメントを生成する
  - DynamoDB への読み書きが正常に動作する
  - JWT 認証が正常に動作する
