# コンポーネント定義 — Sloth Feed

## アーキテクチャ概要

```
app/                        # Next.js App Router ページ
components/                 # 再利用 UI コンポーネント
lib/
  services/                 # ビジネスロジック（薄いコントローラの背後）
  repositories/             # DynamoDB アクセス抽象化
  types/                    # 共有型定義
middleware.ts               # JWT 一括検証
```

---

## バックエンド・コンポーネント

### 1. AuthService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/auth.service.ts` |
| **責務** | ユーザー登録・ログイン・JWT 発行 |
| **依存** | UserRepository, bcrypt, jsonwebtoken |

### 2. PostService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/post.service.ts` |
| **責務** | 投稿作成フローの全体オーケストレーション（フィルタリング → コメント生成 → 保存） |
| **依存** | PostRepository, AIFilteringService, AICommentService |

### 3. FeedService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/feed.service.ts` |
| **責務** | タイムライン取得・自分の投稿一覧取得 |
| **依存** | PostRepository |

### 4. AIFilteringService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/ai-filtering.service.ts` |
| **責務** | Claude API を呼び出し、投稿が「仕事外」かを判定する。除外時は理由も生成 |
| **依存** | Anthropic SDK (@anthropic-ai/sdk) |

### 5. AICommentService
| 項目 | 内容 |
|------|------|
| **パス** | `lib/services/ai-comment.service.ts` |
| **責務** | Claude API を呼び出し、偉人・論文引用付きの称賛コメントを生成する |
| **依存** | Anthropic SDK (@anthropic-ai/sdk) |

### 6. UserRepository
| 項目 | 内容 |
|------|------|
| **パス** | `lib/repositories/user.repository.ts` |
| **責務** | Users テーブルの CRUD（create / findById / findByEmail） |
| **依存** | AWS SDK v3 (@aws-sdk/client-dynamodb, @aws-sdk/lib-dynamodb) |

### 7. PostRepository
| 項目 | 内容 |
|------|------|
| **パス** | `lib/repositories/post.repository.ts` |
| **責務** | Posts テーブルの CRUD + GSI によるユーザー別検索 |
| **依存** | AWS SDK v3 |

---

## フロントエンド・コンポーネント

### ページコンポーネント（`app/`）

| ページ | パス | 説明 |
|--------|------|------|
| タイムライン | `app/(main)/page.tsx` | 全ユーザーの投稿フィード |
| 投稿フォーム | `app/(main)/post/page.tsx` | テキスト入力 → フィルタリング → 結果表示 |
| 自分の投稿 | `app/(main)/my-posts/page.tsx` | 自分の過去投稿一覧 |
| ログイン | `app/auth/login/page.tsx` | ログインフォーム |
| 新規登録 | `app/auth/register/page.tsx` | 新規登録フォーム |

### 再利用 UI コンポーネント（`components/`）

| コンポーネント | パス | 責務 |
|---------------|------|------|
| PostCard | `components/PostCard.tsx` | 投稿本文 + AI称賛コメントを1枚のカードで表示 |
| PostForm | `components/PostForm.tsx` | テキスト入力欄・送信ボタン・バリデーション |
| FeedList | `components/FeedList.tsx` | PostCard の一覧レンダリング（ページネーション対応） |
| AICommentBubble | `components/AICommentBubble.tsx` | AI称賛コメントの吹き出し表示 |
| FilteringFeedback | `components/FilteringFeedback.tsx` | フィルタリング除外時の理由メッセージ表示 |
| AuthForm | `components/AuthForm.tsx` | ログイン・登録共通フォームベース |
| LoadingSpinner | `components/LoadingSpinner.tsx` | Claude API 呼び出し中のローディング表示 |

---

## API Route コンポーネント（`app/api/`）

| エンドポイント | パス | 責務 |
|--------------|------|------|
| POST /api/auth/register | `app/api/auth/register/route.ts` | ユーザー登録 |
| POST /api/auth/login | `app/api/auth/login/route.ts` | ログイン・JWT発行 |
| POST /api/posts | `app/api/posts/route.ts` | 投稿作成（フィルタリング〜コメント生成まで） |
| GET /api/feed | `app/api/feed/route.ts` | タイムライン取得 |
| GET /api/my-posts | `app/api/my-posts/route.ts` | 自分の投稿一覧取得 |

---

## ミドルウェア

| コンポーネント | パス | 責務 |
|--------------|------|------|
| JWTMiddleware | `middleware.ts` | 保護されたルートへのリクエストで JWT を一括検証し、`userId` をリクエストヘッダに付与 |

**保護対象ルート**: `/api/posts`, `/api/my-posts`  
**非保護ルート**: `/api/auth/*`, `/api/feed`（閲覧はログイン不要）
