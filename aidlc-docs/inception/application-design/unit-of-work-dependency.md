# ユニット依存関係マトリクス — Sloth Feed

> **本ドキュメントの位置づけ（2026-05-09 更新・3回目サイクル検証済）**
> 1回目サイクルで作成。3回目サイクルで以下を反映：
> - Unit 2 内に UserHistory（lib/memory/）追加
> - Unit 3 内に BrandFrame（components/）追加
> - **Auth.js + Cognito 移行**：UserRepository を PoC 外、Auth.js + Cognito を外部依存として追加

## 依存関係マトリクス

| 依存元 ↓ / 依存先 → | Unit 1 (Auth) | Unit 2 (Post + AI) | Unit 3 (Feed) |
|------------------|:------------:|:-----------------:|:------------:|
| **Unit 1 (Auth)** | — | なし | なし |
| **Unit 2 (Post + AI)** | **○** | — | なし |
| **Unit 3 (Feed)** | **○** | **○** | — |

- **○** = 依存あり（先に完了している必要がある）

**3回目サイクル更新（2026-05-09）**：
- **Auth.js + Cognito 移行**（中盤の追加）：UserRepository を PoC 外に。authorName は API Route で `session.user.name`（Auth.js Session）から取得して PostService に渡す
- Unit 2 内に `UserHistory`（lib/memory/）が新規追加、Unit 2 内部で完結
- Unit 3 内に `BrandFrame`（components/）が新規追加、Unit 3 内部で完結
- **ユニット間依存マトリクスのセル配置は変更なし**

---

## 依存関係の詳細

### Unit 2 → Unit 1

| 依存内容 | 説明 |
|---------|------|
| `lib/types/index.ts` | `User`, `Post`, `FilterResult`, `CreatePostResult`, `Pathway`, `NamakemonoResponse`, `AuthResult` 等の型定義を利用 |
| `lib/db/client.ts` | DynamoDB クライアントシングルトンを利用 |
| `lib/utils/errors.ts` | 共通エラーハンドリングを利用 |
| ~~`lib/repositories/user.repository.ts`~~ | **3回目サイクル後段で PoC 外**。authorName は API Route が `session.user.name` から取得して PostService に渡す |
| `auth.ts` (Auth.js) | API Route で `await auth()` してセッション取得、`session.user.id` / `session.user.name` を利用 |
| `middleware.ts` | `POST /api/posts` の Auth.js 認証保護に依存（保護ルートとして matcher 設定）|

### Unit 3 → Unit 1

| 依存内容 | 説明 |
|---------|------|
| `lib/types/index.ts` | `Post` 型（`authorName` / `pathway` / `aiCitationSource` 含む）等を利用 |
| `lib/db/client.ts` | DynamoDB クライアントを利用 |
| ~~`lib/repositories/user.repository.ts`~~ | **3回目サイクル後段で PoC 外**。PoC では Post.authorName で表示する（authorId ではない）|
| `auth.ts` (Auth.js) | `/my-posts` ページガード・`GET /api/my-posts` で `await auth()` |
| `middleware.ts` | `GET /api/my-posts` / `/my-posts` の Auth.js 認証保護に依存（保護ルートとして matcher 設定）|

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
| ~~Users テーブル~~ | **3回目サイクルで PoC 外**（Cognito 一本化）|
| ~~`UserRepository`~~ | **3回目サイクルで PoC 外**（Cognito 一本化）|
| **Cognito User Pool** | Unit 1（セットアップ）| Unit 2（API Route が `session.user.name` 取得）/ Unit 3（同上、`/my-posts` ガード）|
| **`auth.ts` (Auth.js)** | Unit 1 | Unit 2（API Route で `await auth()`）/ Unit 3（同上）|
| **`UserHistory`** (`lib/memory/user-history.ts`) | **Unit 2** | Unit 2 内で AINamakemonoService が利用（FR-006 個別化記憶 + FR-009 活動メトリクス）|
| Posts テーブル + GSI | Unit 2 | Unit 3 |
| `PostRepository` | Unit 2 | Unit 3 |
| **`BrandFrame.tsx`** | **Unit 3** | Unit 3 内（サンドイッチUI 上下フレーム）。将来的に他ユニットの UI 拡張時にも再利用可能 |
| `.env.local.example` | Unit 1 | Unit 1 で Cognito + Auth.js 系（`COGNITO_USER_POOL_ID`/`COGNITO_APP_CLIENT_ID`/`COGNITO_APP_CLIENT_SECRET`/`AUTH_SECRET`/`NEXTAUTH_URL`）を追加。Unit 2 で `BEDROCK_MODEL_ID`（および IAM 関連設定）を追加 |

---

## 外部依存関係

| 外部サービス | 利用ユニット | 依存内容 |
|------------|------------|---------|
| AWS DynamoDB | Unit 1, 2, 3 | テーブル読み書き |
| Amazon Bedrock（Claude モデル） | Unit 2 のみ | フィルタリング判定・コメント生成（`@aws-sdk/client-bedrock-runtime` 経由）|
| **AWS Cognito User Pool** | Unit 1 (セットアップ) / Unit 2, 3 (API Route で参照) | ユーザー登録・ログイン・パスワード管理・JWT 発行（OAuth/OIDC 経由）|
| **Auth.js (NextAuth v5)** | Unit 1 (設定) / Unit 2, 3 (`await auth()` で利用) | 認証フレームワーク。Cognito Provider 経由で OAuth/OIDC、HttpOnly Cookie セッション管理、JWT 検証（JWKS 自動取得）|
