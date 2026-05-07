# コンポーネント・メソッドシグネチャ — Sloth Feed

> 詳細なビジネスロジック（Claude プロンプト文面・DynamoDB クエリ最適化等）は
> コンストラクション・フェーズの機能設計 (Functional Design) で定義する。

---

## 共有型定義（`lib/types/index.ts`）

```typescript
type User = {
  userId: string;
  email: string;
  passwordHash: string;
  createdAt: string; // ISO 8601
};

type Post = {
  postId: string;
  content: string;
  authorId: string;
  aiComment: string;
  createdAt: string; // ISO 8601
};

type FilterResult =
  | { allowed: true }
  | { allowed: false; reason: string };

type CreatePostResult =
  | { success: true; post: Post }
  | { success: false; reason: string };

type PaginatedResult<T> = {
  items: T[];
  lastKey?: string; // DynamoDB ExclusiveStartKey のシリアライズ値
};

type AuthResult = {
  userId: string;
  token: string; // JWT
};
```

---

## AuthService

```typescript
class AuthService {
  // ユーザー登録。メール重複時はエラーをスロー
  register(email: string, password: string): Promise<AuthResult>;

  // ログイン。認証失敗時はエラーをスロー
  login(email: string, password: string): Promise<AuthResult>;
}
```

---

## PostService

```typescript
class PostService {
  // 投稿作成の全フロー（フィルタリング → コメント生成 → 保存）を実行
  // フィルタリングで除外された場合は success: false と除外理由を返す
  createPost(authorId: string, content: string): Promise<CreatePostResult>;
}
```

---

## FeedService

```typescript
class FeedService {
  // 全ユーザーの投稿を新しい順で取得（タイムライン）
  getTimeline(limit: number, lastKey?: string): Promise<PaginatedResult<Post>>;

  // 指定ユーザーの投稿を新しい順で取得（自分の投稿一覧）
  getUserPosts(authorId: string, limit: number, lastKey?: string): Promise<PaginatedResult<Post>>;
}
```

---

## AIFilteringService

```typescript
class AIFilteringService {
  // Claude API で投稿内容を判定する
  // 仕事の成果・旅行・スポーツ大会結果等は allowed: false + reason を返す
  filterPost(content: string): Promise<FilterResult>;
}
```

---

## AICommentService

```typescript
class AICommentService {
  // Claude API で称賛コメントを生成する
  // 偉人の名言・研究論文・心理学的知見のいずれかを引用した形式で返す
  generateComment(content: string): Promise<string>;
}
```

---

## UserRepository

```typescript
class UserRepository {
  // メールアドレスでユーザーを検索（ログイン・重複チェックに使用）
  findByEmail(email: string): Promise<User | null>;

  // ユーザーID でユーザーを検索
  findById(userId: string): Promise<User | null>;

  // 新規ユーザーを作成
  create(input: { email: string; passwordHash: string }): Promise<User>;
}
```

---

## PostRepository

```typescript
class PostRepository {
  // 新規投稿を保存
  create(input: {
    content: string;
    authorId: string;
    aiComment: string;
  }): Promise<Post>;

  // 全投稿を createdAt 降順で取得（タイムライン用）
  findAll(limit: number, lastKey?: string): Promise<PaginatedResult<Post>>;

  // 指定 authorId の投稿を createdAt 降順で取得（自分の投稿一覧用）
  findByAuthorId(
    authorId: string,
    limit: number,
    lastKey?: string
  ): Promise<PaginatedResult<Post>>;
}
```

---

## API Route ハンドラ（薄いコントローラ）

```typescript
// POST /api/auth/register
// Body: { email: string; password: string }
// Response 201: { userId: string; token: string }

// POST /api/auth/login
// Body: { email: string; password: string }
// Response 200: { userId: string; token: string }

// POST /api/posts
// Headers: Authorization: Bearer <token>  ← middleware.ts で検証済み
// Body: { content: string }
// Response 201: { post: Post }
// Response 422: { reason: string }  ← フィルタリング除外

// GET /api/feed?limit=20&lastKey=xxx
// Response 200: { items: Post[]; lastKey?: string }

// GET /api/my-posts?limit=20&lastKey=xxx
// Headers: Authorization: Bearer <token>  ← middleware.ts で検証済み
// Response 200: { items: Post[]; lastKey?: string }
```

---

## middleware.ts

```typescript
// 保護対象パス: /api/posts, /api/my-posts
// JWT を Authorization ヘッダから取得・検証
// 検証成功: x-user-id ヘッダに userId を付与して次のハンドラへ
// 検証失敗: 401 Unauthorized を返す
export function middleware(request: NextRequest): NextResponse;
```
