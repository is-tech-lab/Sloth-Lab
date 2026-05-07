# アプリケーション設計 — Sloth Feed PoC

## 設計サマリー

| 項目 | 決定内容 |
|------|---------|
| フレームワーク | Next.js 14+ App Router / TypeScript |
| サービスレイヤー | `lib/services/` に分離（薄いコントローラ + サービスクラス） |
| DBアクセス | リポジトリパターン（`lib/repositories/`） |
| DynamoDB | テーブル分割（Users / Posts） |
| AI サービス | AIFilteringService と AICommentService を分離 |
| JWT 認証 | `middleware.ts` で一括検証 |
| UIコンポーネント | `app/`（ページ）+ `components/`（再利用 UI）分離 |

---

## ディレクトリ構造

```
sloth-feed/
├── app/
│   ├── (main)/
│   │   ├── page.tsx                   # タイムライン
│   │   ├── post/
│   │   │   └── page.tsx               # 投稿フォーム
│   │   └── my-posts/
│   │       └── page.tsx               # 自分の投稿一覧
│   ├── auth/
│   │   ├── login/
│   │   │   └── page.tsx               # ログイン
│   │   └── register/
│   │       └── page.tsx               # 新規登録
│   └── api/
│       ├── auth/
│       │   ├── register/
│       │   │   └── route.ts
│       │   └── login/
│       │       └── route.ts
│       ├── posts/
│       │   └── route.ts               # POST: 投稿作成
│       ├── feed/
│       │   └── route.ts               # GET: タイムライン
│       └── my-posts/
│           └── route.ts               # GET: 自分の投稿一覧
├── components/
│   ├── PostCard.tsx
│   ├── PostForm.tsx
│   ├── FeedList.tsx
│   ├── AICommentBubble.tsx
│   ├── FilteringFeedback.tsx
│   ├── AuthForm.tsx
│   └── LoadingSpinner.tsx
├── lib/
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── post.service.ts
│   │   ├── feed.service.ts
│   │   ├── ai-filtering.service.ts
│   │   └── ai-comment.service.ts
│   ├── repositories/
│   │   ├── user.repository.ts
│   │   └── post.repository.ts
│   └── types/
│       └── index.ts
└── middleware.ts
```

---

## コンポーネント一覧

### バックエンド・サービス

| コンポーネント | パス | 主な責務 |
|--------------|------|---------|
| AuthService | `lib/services/auth.service.ts` | 登録・ログイン・JWT発行 |
| PostService | `lib/services/post.service.ts` | 投稿作成フローのオーケストレーション |
| FeedService | `lib/services/feed.service.ts` | タイムライン・自分の投稿取得 |
| AIFilteringService | `lib/services/ai-filtering.service.ts` | Claude API でフィルタリング判定 |
| AICommentService | `lib/services/ai-comment.service.ts` | Claude API で称賛コメント生成 |
| UserRepository | `lib/repositories/user.repository.ts` | Users テーブル CRUD |
| PostRepository | `lib/repositories/post.repository.ts` | Posts テーブル CRUD + GSI 検索 |
| middleware.ts | `middleware.ts` | 保護ルートの JWT 一括検証 |

### フロントエンド・UIコンポーネント

| コンポーネント | 説明 |
|--------------|------|
| PostCard | 投稿本文 + AI称賛コメントを1枚で表示 |
| PostForm | テキスト入力・送信・バリデーション |
| FeedList | PostCard のリスト（ページネーション対応） |
| AICommentBubble | AI称賛コメントの吹き出し表示 |
| FilteringFeedback | フィルタリング除外時の理由表示 |
| AuthForm | ログイン・登録の共通フォームベース |
| LoadingSpinner | Claude API 呼び出し中のローディング |

---

## コアフロー：投稿作成

```
クライアント → middleware.ts（JWT検証）
             → POST /api/posts
             → PostService.createPost
                  → AIFilteringService.filterPost（Claude API）
                       ├─ 除外: 422 + reason を返す
                       └─ 通過:
                  → AICommentService.generateComment（Claude API）
                  → PostRepository.create（DynamoDB）
                  → 201 + Post を返す
```

---

## DynamoDB テーブル設計

### Users テーブル
| 属性 | 型 | キー |
|------|----|------|
| userId | String | PK |
| email | String | GSI PK (email-index) |
| passwordHash | String | |
| createdAt | String (ISO) | |

### Posts テーブル
| 属性 | 型 | キー |
|------|----|------|
| postId | String | PK |
| content | String | |
| authorId | String | GSI PK (authorId-createdAt-index) |
| aiComment | String | |
| createdAt | String (ISO) | GSI SK |

---

## API エンドポイント一覧

| メソッド | パス | 認証 | 説明 |
|---------|------|------|------|
| POST | /api/auth/register | なし | ユーザー登録 |
| POST | /api/auth/login | なし | ログイン・JWT発行 |
| POST | /api/posts | JWT | 投稿作成（フィルタリング〜コメント生成） |
| GET | /api/feed | なし | タイムライン取得 |
| GET | /api/my-posts | JWT | 自分の投稿一覧取得 |

---

## 環境変数

| 変数名 | 用途 |
|--------|------|
| ANTHROPIC_API_KEY | Claude API 認証 |
| DYNAMODB_USERS_TABLE | Users テーブル名 |
| DYNAMODB_POSTS_TABLE | Posts テーブル名 |
| AWS_REGION | DynamoDB リージョン |
| JWT_SECRET | JWT 署名シークレット |
| JWT_EXPIRES_IN | JWT 有効期限（例: `7d`） |

---

## 設計の詳細

各成果物の詳細はそれぞれのファイルを参照：

- コンポーネント定義・責務 → [components.md](components.md)
- メソッドシグネチャ・型定義 → [component-methods.md](component-methods.md)
- サービス定義・オーケストレーション → [services.md](services.md)
- 依存関係・データフロー図 → [component-dependency.md](component-dependency.md)
