# ユニット・オブ・ワーク定義 — Sloth Feed

## 変更点（実行計画からの更新）

実行計画では4ユニット（Auth / Post+Filtering / AIComment / Feed）を予定していたが、
ユニット生成フェーズでの決定により **Unit 2 と Unit 3 を統合**。
最終構成は **3ユニット** となる。

---

## ユニット一覧

| Unit | 名称 | 開発順序 |
|------|------|---------|
| Unit 1 | 認証（Auth） | 1番目 |
| Unit 2 | 投稿 + AI（Post + AI） | 2番目 |
| Unit 3 | フィード（Feed） | 3番目 |

---

## Unit 1 — 認証（Auth）

**スコープ**: Next.js プロジェクト初期化 + メール・パスワード認証 + 共通基盤

### 含まれる機能
- Next.js 14+ App Router プロジェクトのスキャフォールディング
- メールアドレス＋パスワードによるユーザー登録
- ログイン・JWT 発行・JWT 検証ミドルウェア
- DynamoDB Users テーブルの作成
- `lib/` 共通モジュールの初期化（後続ユニットで再利用）

### 生成するファイル

**プロジェクト基盤**
```
package.json
tsconfig.json
next.config.ts
.env.local.example
```

**共通モジュール（後続ユニットが再利用）**
```
lib/types/index.ts            # 共有型定義（User, Post, FilterResult 等）
lib/db/client.ts              # DynamoDB クライアントシングルトン
lib/utils/errors.ts           # エラーハンドリングユーティリティ
```

**Auth モジュール**
```
lib/repositories/user.repository.ts
lib/services/auth.service.ts
app/api/auth/register/route.ts
app/api/auth/login/route.ts
app/auth/login/page.tsx
app/auth/register/page.tsx
components/AuthForm.tsx
middleware.ts
```

### DynamoDB テーブル
- **Users** テーブル（PK: userId、GSI: email-index）

### 完了基準
- `POST /api/auth/register` でユーザーを作成できる
- `POST /api/auth/login` で JWT を取得できる
- `middleware.ts` が保護ルートで JWT を検証し、`x-user-id` ヘッダを付与する

---

## Unit 2 — 投稿 + AI（Post + AI）

**スコープ**: テキスト投稿フロー全体（フィルタリング → コメント生成 → 保存）

### 含まれる機能
- 投稿テキスト入力フォーム
- Claude API によるフィルタリング判定（仕事の成果・旅行・スポーツ大会結果等を除外）
- フィルタリング除外時の理由表示（Claude が日本語で生成）
- フィルタリング通過後の Claude API による称賛コメント生成
- 投稿 + AI コメントの DynamoDB 保存

### 前提条件
- Unit 1 完了（認証・`lib/` 共通モジュール・DynamoDB クライアント利用可能）

### 生成するファイル
```
lib/repositories/post.repository.ts
lib/services/ai-filtering.service.ts
lib/services/ai-comment.service.ts
lib/services/post.service.ts
app/api/posts/route.ts
app/(main)/post/page.tsx
components/PostForm.tsx
components/FilteringFeedback.tsx
components/AICommentBubble.tsx
components/LoadingSpinner.tsx
```

### DynamoDB テーブル
- **Posts** テーブル（PK: postId、GSI: authorId-createdAt-index）

### 完了基準
- `POST /api/posts` に `content` を送ると Claude がフィルタリング判定する
- 仕事の成果・旅行投稿が 422（除外理由付き）で返される
- 「仕事じゃないけど」系の投稿が通過し、AI 称賛コメントとともに保存される

---

## Unit 3 — フィード（Feed）

**スコープ**: タイムライン閲覧・自分の投稿一覧閲覧

### 含まれる機能
- 全ユーザーの投稿タイムライン（新しい順、未ログインで閲覧可能）
- 自分の過去投稿一覧（ログイン必須）
- 各投稿に AI 称賛コメントを表示

### 前提条件
- Unit 1 完了（認証・`lib/` 共通モジュール）
- Unit 2 完了（PostRepository・Posts テーブル利用可能）

### 生成するファイル
```
lib/services/feed.service.ts
app/api/feed/route.ts
app/api/my-posts/route.ts
app/(main)/page.tsx           # タイムライン（未ログインでも閲覧可）
app/(main)/my-posts/page.tsx  # 自分の投稿一覧（ログイン必須）
components/PostCard.tsx
components/FeedList.tsx
```

### 完了基準
- `GET /api/feed` が未認証でもタイムラインを返す
- `GET /api/my-posts` が JWT 必須で自分の投稿一覧を返す
- タイムラインページ（`/`）が未ログインユーザーでもアクセス可能

---

## 実行順序

```
Unit 1（Auth）
  └─→ Unit 2（Post + AI）        ← Unit 1 の lib/ 共通モジュールを利用
          └─→ Unit 3（Feed）      ← Unit 1・2 の Repository・テーブルを利用
```

## コード整理戦略

すべてのユニットは **1 つの Next.js モノリポジトリ** 内に配置する。

| フォルダ | 所有ユニット | 説明 |
|---------|------------|------|
| `lib/types/` | Unit 1 で初期化、全ユニット利用 | 共有型定義 |
| `lib/db/` | Unit 1 で初期化、全ユニット利用 | DynamoDB クライアント |
| `lib/utils/` | Unit 1 で初期化、全ユニット利用 | 共通ユーティリティ |
| `lib/repositories/user.repository.ts` | Unit 1 | Users テーブルアクセス |
| `lib/services/auth.service.ts` | Unit 1 | 認証ロジック |
| `lib/repositories/post.repository.ts` | Unit 2 | Posts テーブルアクセス |
| `lib/services/ai-filtering.service.ts` | Unit 2 | フィルタリング |
| `lib/services/ai-comment.service.ts` | Unit 2 | コメント生成 |
| `lib/services/post.service.ts` | Unit 2 | 投稿オーケストレーション |
| `lib/services/feed.service.ts` | Unit 3 | フィード取得 |
