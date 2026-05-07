# ユニット依存関係マトリクス — Sloth Feed

## 依存関係マトリクス

| 依存元 ↓ / 依存先 → | Unit 1 (Auth) | Unit 2 (Post + AI) | Unit 3 (Feed) |
|------------------|:------------:|:-----------------:|:------------:|
| **Unit 1 (Auth)** | — | なし | なし |
| **Unit 2 (Post + AI)** | **○** | — | なし |
| **Unit 3 (Feed)** | **○** | **○** | — |

- **○** = 依存あり（先に完了している必要がある）

---

## 依存関係の詳細

### Unit 2 → Unit 1

| 依存内容 | 説明 |
|---------|------|
| `lib/types/index.ts` | `User`, `Post`, `FilterResult`, `CreatePostResult` 等の型定義を利用 |
| `lib/db/client.ts` | DynamoDB クライアントシングルトンを利用 |
| `lib/utils/errors.ts` | 共通エラーハンドリングを利用 |
| `middleware.ts` | `POST /api/posts` の JWT 認証に依存（`x-user-id` ヘッダを受け取る） |

### Unit 3 → Unit 1

| 依存内容 | 説明 |
|---------|------|
| `lib/types/index.ts` | `Post` 型等を利用 |
| `lib/db/client.ts` | DynamoDB クライアントを利用 |
| `middleware.ts` | `GET /api/my-posts` の JWT 認証に依存 |

### Unit 3 → Unit 2

| 依存内容 | 説明 |
|---------|------|
| `lib/repositories/post.repository.ts` | FeedService が PostRepository を利用 |
| DynamoDB Posts テーブル | Unit 2 で作成したテーブル・GSI に依存 |

---

## ビルド順序の強制

```
Unit 1 完了
  ↓
Unit 2 開始可能
  ↓
Unit 2 完了
  ↓
Unit 3 開始可能
```

**Unit 2 を Unit 1 より先に開始することはできない**：DynamoDB クライアントと型定義が未定義  
**Unit 3 を Unit 2 より先に開始することはできない**：PostRepository と Posts テーブルが未作成

---

## ユニット間の共有リソース

| リソース | 初期化ユニット | 利用ユニット |
|---------|--------------|------------|
| DynamoDB クライアント (`lib/db/client.ts`) | Unit 1 | Unit 2, Unit 3 |
| 共有型定義 (`lib/types/index.ts`) | Unit 1 | Unit 2, Unit 3 |
| エラーユーティリティ (`lib/utils/errors.ts`) | Unit 1 | Unit 2, Unit 3 |
| Users テーブル | Unit 1 | — |
| `UserRepository` | Unit 1 | — （今後ログイン中ユーザー情報表示が必要になれば Unit 3 も利用） |
| Posts テーブル + GSI | Unit 2 | Unit 3 |
| `PostRepository` | Unit 2 | Unit 3 |
| `.env.local.example` | Unit 1 | Unit 2 で `ANTHROPIC_API_KEY` を追加 |

---

## 外部依存関係

| 外部サービス | 利用ユニット | 依存内容 |
|------------|------------|---------|
| AWS DynamoDB | Unit 1, 2, 3 | テーブル読み書き |
| Claude API (Anthropic) | Unit 2 のみ | フィルタリング判定・コメント生成 |
| bcrypt | Unit 1 のみ | パスワードハッシュ |
| jsonwebtoken | Unit 1, middleware | JWT 発行・検証 |
