# コンポーネント依存関係 — Sloth Feed

## 依存関係マトリクス

| コンポーネント | AuthService | PostService | FeedService | AIFilteringService | AICommentService | UserRepository | PostRepository | Claude API | DynamoDB |
|-------------|:-----------:|:-----------:|:-----------:|:-----------------:|:----------------:|:--------------:|:--------------:|:----------:|:--------:|
| API Route (auth) | ○ | | | | | | | | |
| API Route (posts) | | ○ | | | | | | | |
| API Route (feed/my-posts) | | | ○ | | | | | | |
| AuthService | | | | | | ○ | | | |
| PostService | | | | ○ | ○ | | ○ | | |
| FeedService | | | | | | | ○ | | |
| AIFilteringService | | | | | | | | ○ | |
| AICommentService | | | | | | | | ○ | |
| UserRepository | | | | | | | | | ○ |
| PostRepository | | | | | | | | | ○ |
| middleware.ts | | | | | | | | | |

---

## データフロー図

### フロー 1: 投稿作成（コアフロー）

```
クライアント
  │  POST /api/posts  { content }
  │  Authorization: Bearer <token>
  ↓
middleware.ts
  │  JWT 検証 → x-user-id ヘッダ付与
  ↓
API Route: /api/posts/route.ts
  │  authorId = request.headers['x-user-id']
  ↓
PostService.createPost(authorId, content)
  │
  ├──→ AIFilteringService.filterPost(content)
  │         │  Claude API 呼び出し
  │         └─ FilterResult
  │              ├─ allowed: false
  │              │     └── { success: false, reason }
  │              │              → API Route が 422 を返す
  │              └─ allowed: true
  │                    ↓
  ├──→ AICommentService.generateComment(content)
  │         │  Claude API 呼び出し
  │         └─ aiComment: string
  │
  └──→ PostRepository.create({ content, authorId, aiComment })
            │  DynamoDB Posts テーブルに書き込み
            └─ Post
                 → API Route が 201 を返す
```

### フロー 2: ユーザー登録

```
クライアント
  │  POST /api/auth/register  { email, password }
  ↓
API Route: /api/auth/register/route.ts
  ↓
AuthService.register(email, password)
  │
  ├──→ UserRepository.findByEmail(email)  ← 重複チェック
  │         DynamoDB Users テーブル検索
  │
  ├── bcrypt.hash(password)
  │
  ├──→ UserRepository.create({ email, passwordHash })
  │         DynamoDB Users テーブルに書き込み
  │
  └── JWT 発行
       → API Route が 201 { userId, token } を返す
```

### フロー 3: タイムライン取得

```
クライアント
  │  GET /api/feed?limit=20&lastKey=xxx
  ↓
API Route: /api/feed/route.ts  ← 認証不要
  ↓
FeedService.getTimeline(limit, lastKey?)
  ↓
PostRepository.findAll(limit, lastKey?)
  │  DynamoDB Posts テーブルを createdAt 降順でスキャン
  └─ PaginatedResult<Post>
       → API Route が 200 を返す
```

### フロー 4: 自分の投稿一覧

```
クライアント
  │  GET /api/my-posts?limit=20&lastKey=xxx
  │  Authorization: Bearer <token>
  ↓
middleware.ts
  │  JWT 検証 → x-user-id ヘッダ付与
  ↓
API Route: /api/my-posts/route.ts
  ↓
FeedService.getUserPosts(authorId, limit, lastKey?)
  ↓
PostRepository.findByAuthorId(authorId, limit, lastKey?)
  │  DynamoDB Posts テーブルを GSI (authorId-createdAt-index) で検索
  └─ PaginatedResult<Post>
       → API Route が 200 を返す
```

---

## コミュニケーションパターン

| パターン | 採用箇所 | 説明 |
|---------|---------|------|
| 同期呼び出し | すべてのサービス間 | async/await による直列・直接呼び出し |
| リポジトリパターン | Service → Repository | DynamoDB の実装詳細をサービスから隠蔽 |
| 薄いコントローラ | API Route → Service | HTTP の入出力変換のみ担当 |
| JWT ミドルウェア | middleware.ts → API Route | 認証済み userId を後続ハンドラに注入 |
| 外部API呼び出し | AIFilteringService / AICommentService → Claude API | Anthropic SDK 経由の HTTP 呼び出し |

---

## DynamoDB テーブル設計

### Users テーブル

| キー | 型 | 説明 |
|------|----|------|
| userId (PK) | String | UUID v4 |
| email | String | メールアドレス（GSI: email-index の PK） |
| passwordHash | String | bcrypt ハッシュ |
| createdAt | String | ISO 8601 |

- **GSI**: `email-index`（PK: email）— ログイン時の findByEmail に使用

### Posts テーブル

| キー | 型 | 説明 |
|------|----|------|
| postId (PK) | String | UUID v4 |
| content | String | 投稿本文 |
| authorId | String | 投稿者の userId |
| aiComment | String | AI生成の称賛コメント |
| createdAt | String | ISO 8601（ソート用） |

- **GSI**: `authorId-createdAt-index`（PK: authorId, SK: createdAt）— getUserPosts に使用
- タイムライン（全件 createdAt 降順）は Scan + フィルタ（PoC 許容）
