# コンポーネント・メソッドシグネチャ — Sloth Feed

> 最新版：2026-05-09 / 改訂履歴は [`audit.md`](../../audit.md) と [`aidlc-state.md`](../../aidlc-state.md) を参照。

> 詳細なビジネスロジック（Claude プロンプト文面・DynamoDB クエリ最適化等）は
> コンストラクション・フェーズの機能設計 (Functional Design) で定義する。具体的には：System Prompt（老師人格 / FR-010）・5経路選択ロジック・連続投稿/滞在時間検知（依存防止 / FR-009）・引用源プロンプトヒント（FR-007）・経路ラベル付加（FR-011）など。

---

## 共有型定義（`lib/types/index.ts`）

```typescript
type User = {
  userId: string;
  email: string;
  name: string;              // 表示名（FR-001 投稿UI・US-007 タイムライン表示）
  passwordHash: string;
  createdAt: string; // ISO 8601
};

type Post = {
  postId: string;
  content: string;
  authorId: string;
  authorName: string;        // 投稿時の User.name をスナップショット保存（denormalization）
  aiComment: string;
  aiCitationSource: string; // 引用元（例：「Larry Wall」「ラッセル『怠惰への讃歌』」）
  pathway: Pathway;          // 5経路のうち紐付けられたもの（FR-011）
  createdAt: string; // ISO 8601
};

type FilterResult =
  | { allowed: true }
  | { allowed: false; reason: string };

// 5経路（FR-003 / FR-011）
type Pathway = 1 | 2 | 3 | 4 | 5;
type PathwayLabel =
  | "①過剰生産社会へのブレーキ"
  | "②創造の余白の保持"
  | "③多様性の保護"
  | "④自己への暴力の停止"
  | "⑤集積による文化変容";

// AI ナマケモノの応答（FR-003 / FR-007 / FR-011）
type NamakemonoResponse = {
  pathway: Pathway;          // 5経路のうち選択したもの
  pathwayLabel: PathwayLabel; // 経路ラベル表示用
  comment: string;            // 老師人格による肯定コメント本文
  citationSource: string;     // 引用元（PoC では LLM 自己申告）
  shouldSuggestBreak: boolean; // FR-009 切り上げ提案フラグ
};

type CreatePostResult =
  | { success: true; post: Post }
  | { success: false; failureType: 'filtering_excluded'; message: string }     // FR-002 仕事系除外（HTTP 422）
  | { success: false; failureType: 'ai_generation_failed'; message: string }   // Bedrock タイムアウト・エラー（HTTP 502、再投稿可）
  | { success: false; failureType: 'persistence_failed'; message: string };    // DynamoDB 書き込み失敗（HTTP 500、再投稿可）

// 失敗時の対応方針：
// - filtering_excluded: タイムラインに残らない（仕様通り）
// - ai_generation_failed: AI 応答未生成のため保存しない、ユーザーに再投稿を促す
// - persistence_failed: AI 応答は揮発、ユーザーに再投稿を促す
// 詳細なリトライポリシー・冪等性キーは機能設計で定義

type PaginatedResult<T> = {
  items: T[];
  lastKey?: string; // DynamoDB ExclusiveStartKey のシリアライズ値
};

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
  // Amazon Bedrock 経由で Claude モデルを呼び出して投稿内容を判定する
  // 仕事の成果・旅行・スポーツ大会結果等は allowed: false + reason を返す
  filterPost(content: string): Promise<FilterResult>;
}
```

---

## AINamakemonoService（**動的IPの核**）

```typescript
class AINamakemonoService {
  // 投稿に対して AI ナマケモノが応答する（動的IPの核）
  // - **5経路のいずれかに紐付け**（FR-003）
  // - **「達観した怠惰の老師」人格**で生成（FR-010）
  // - **LLM の学習済み知識**から偉人・科学・歴史の引用を生成（PoC：FR-007）
  // - **個別化記憶**（過去投稿）を参照して継続性を持たせる（FR-006）
  // - **依存防止判定**：連続投稿/滞在時間に基づき切り上げ提案フラグを返す（FR-009）
  //
  // 注：詳細な System Prompt 文面・5経路選択ロジック・閾値チューニングは
  // 機能設計 (Functional Design) で定義する。
  generateResponse(authorId: string, content: string): Promise<NamakemonoResponse>;
}
```

---

## UserHistory（個別化記憶）

```typescript
class UserHistory {
  // ユーザーの過去投稿（最近 N 件）を取得
  // AINamakemonoService が System Prompt 構築時に参照（FR-006）
  getRecent(authorId: string, limit: number): Promise<Post[]>;

  // 連続使用時間・連続投稿数を取得（FR-009 依存防止判定用）
  getActivityMetrics(authorId: string): Promise<{
    consecutivePosts: number;
    sessionMinutes: number;
  }>;
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
    authorName: string;        // 投稿時の User.name をスナップショット（denormalization）
    aiComment: string;
    aiCitationSource: string; // 引用元（FR-007）
    pathway: Pathway;          // 5経路のうちどれに紐付いたか（FR-011・将来の経路分布機能の前提）
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
// ── 認証関連エンドポイント ──
// Auth.js のハンドラ /api/auth/[...nextauth]/route.ts に統一（Cognito OAuth フロー）
// 詳細は本ドキュメント末尾の「Authentication & Identity Flow」セクション参照

// POST /api/posts
// Cookie: Auth.js セッション Cookie（HttpOnly、middleware が検証）
// Body: { content: string }
// Response 201: { post: Post }
// Response 422: { failureType: 'filtering_excluded'; message: string }
// Response 502: { failureType: 'ai_generation_failed'; message: string }
// Response 500: { failureType: 'persistence_failed'; message: string }

// GET /api/feed?limit=20&lastKey=xxx
// Response 200: { items: Post[]; lastKey?: string }

// GET /api/my-posts?limit=20&lastKey=xxx
// Cookie: Auth.js セッション Cookie（middleware が検証）
// Response 200: { items: Post[]; lastKey?: string }
```

---

## middleware.ts（Auth.js 委譲）

```typescript
// Auth.js の auth ヘルパに保護機構を委譲する。
//
// import { auth } from "@/auth"
// export default auth
// export const config = {
//   matcher: ["/api/posts/:path*", "/api/my-posts/:path*", "/post", "/my-posts"],
// }
//
// 検証成功: Auth.js が Cookie を解読してセッションを構築。API Route 側で `await auth()` でアクセス
// 検証失敗: Auth.js が 401 / リダイレクトを返す（matcher 設定に応じて）
//
// 詳細は本ドキュメント末尾の「Authentication & Identity Flow」セクション参照
```

---

## lib/utils/errors.ts（共通エラー型）

```typescript
/**
 * アプリケーション全体で使う統一エラー型
 * - statusCode: HTTP ステータスコード
 * - failureType: CreatePostResult.failureType 等の機械可読な分類
 * - message: ユーザー向けの日本語メッセージ
 */
class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly failureType: string,
    message: string
  ) {
    super(message);
    this.name = 'AppError';
  }
}

/**
 * 型ガード：unknown が AppError かを判定
 * Bedrock や DynamoDB の SDK エラーと AppError を分けて扱う際に使用
 */
function isAppError(err: unknown): err is AppError {
  return err instanceof AppError;
}

/**
 * よく使うエラー生成ヘルパ（PoC で共通の HTTP マッピングをそろえるため）
 */
function filteringExcludedError(message: string): AppError {
  return new AppError(422, 'filtering_excluded', message);
}

function aiGenerationFailedError(message: string): AppError {
  return new AppError(502, 'ai_generation_failed', message);
}

function persistenceFailedError(message: string): AppError {
  return new AppError(500, 'persistence_failed', message);
}

function unauthorizedError(message: string): AppError {
  return new AppError(401, 'unauthorized', message);
}

function validationError(message: string): AppError {
  return new AppError(400, 'validation_failed', message);
}
```

**設計意図**：
- **統一型**：API Route 層は `isAppError(err)` でアプリ既知エラー / 未知エラーを分岐し、未知は 500 を返す
- **CreatePostResult.failureType と整合**：filtering_excluded / ai_generation_failed / persistence_failed の3種類は `failureType` フィールドにそのまま対応
- **詳細実装は機能設計で**：ロギング戦略・スタックトレースの扱い・エラー監視（CloudWatch）連携は機能設計で確定

---

## バリデーション要件（PoC）

| フィールド | 制約 | 検証場所 |
|---|---|---|
| email | RFC 5322 簡易形式 + **Cognito ポリシー**| Cognito 側で一意性チェック |
| password | **Cognito ポリシー**（PoC: 最低 8 文字、機能設計で詳細化）| Cognito 側で検証 |
| name | 1〜20 文字（Cognito カスタム属性 `custom:name`）| Cognito 登録時に検証 |
| content（投稿）| 1〜500 文字（PoC）| API Route 入力検証・FR-001 |

詳細なルール（特殊文字制限・Unicode 正規化・絵文字許容範囲等）は機能設計で確定する。

---

## Authentication & Identity Flow (PoC) — Auth.js + AWS Cognito

### スタック決定

| レイヤー | 技術 |
|---|---|
| 認証フレームワーク | **Auth.js (NextAuth v5)** |
| ID プロバイダ | **AWS Cognito User Pool**（OAuth/OIDC プロバイダとして接続）|
| セッション保存 | **HttpOnly Cookie**（Auth.js デフォルト、XSS 耐性 ◎）|
| パスワード管理 | **Cognito**（Sloth Feed 側で passwordHash を持たない）|
| JWT 発行 / 検証 | **Cognito** が発行、**Auth.js** が検証（JWKS 自動取得）|

### UX の遅延決定

**登録 / ログイン画面の UI 形態は PoC 実装時に決定**：

- Option A: **Cognito Hosted UI**（Cognito 提供のリダイレクト先ページ）
- Option B: **自前フォーム**（Auth.js の Credentials Provider または Cognito の SignUp/InitiateAuth API を直接叩く）

→ PoC 実装着手時に再評価。**設計時点ではどちらでも対応可能なように Auth.js を中心に組む**。

### 主要コンポーネント

#### `auth.ts`（プロジェクトルート）— Auth.js 設定の中心

```typescript
import NextAuth from "next-auth"
import Cognito from "next-auth/providers/cognito"

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Cognito({
      clientId: process.env.COGNITO_APP_CLIENT_ID!,
      clientSecret: process.env.COGNITO_APP_CLIENT_SECRET!,
      issuer: `https://cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}`,
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      session.user.id = token.sub as string;
      session.user.name = (token['custom:name'] as string) ?? token.name;
      return session;
    },
  },
})
```

#### `app/api/auth/[...nextauth]/route.ts` — Auth.js のルートハンドラ

```typescript
export { GET, POST } from "@/auth";
```

#### `middleware.ts` — Auth.js の保護機構を利用

```typescript
import { auth } from "@/auth";
export default auth;
export const config = {
  matcher: ["/api/posts/:path*", "/api/my-posts/:path*", "/post", "/my-posts"],
};
```

→ JWKS の検証・iss/aud/exp 検証は **Auth.js が内部で実施**。Sloth Feed 側で書く必要なし。

### Session 型（Auth.js 標準を拡張）

```typescript
import "next-auth";

declare module "next-auth" {
  interface Session {
    user: {
      id: string;     // Cognito sub
      name: string;   // Cognito custom:name
      email?: string;
    };
  }
}
```

### API Route での認証情報取得

```typescript
import { auth } from "@/auth";

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user?.id) {
    return new Response('Unauthorized', { status: 401 });
  }
  const userId = session.user.id;       // Cognito sub
  const userName = session.user.name;    // Cognito custom:name
  // ... PostService.createPost(userId, userName, content) など
}
```

→ **PostService に authorName を直接渡せる**。UserRepository.findById は不要（Cognito 直結のため）。

### クライアントでの認証状態

```typescript
'use client';
import { useSession, signIn, signOut } from "next-auth/react";

export function Header() {
  const { data: session, status } = useSession();
  if (status === "loading") return null;
  if (!session) return <button onClick={() => signIn("cognito")}>ログイン</button>;
  return (
    <>
      こんにちは、{session.user.name} さん
      <button onClick={() => signOut()}>ログアウト</button>
    </>
  );
}
```

### 環境変数（Cognito + Auth.js）

| 変数名 | 用途 |
|---|---|
| `COGNITO_USER_POOL_ID` | Cognito ユーザープール ID |
| `COGNITO_APP_CLIENT_ID` | App Client ID |
| `COGNITO_APP_CLIENT_SECRET` | App Client Secret（Auth.js OAuth が要求）|
| `AUTH_SECRET` | Auth.js Cookie 暗号化用ランダム文字列 |
| `NEXTAUTH_URL` | アプリ Base URL（例: `http://localhost:3000`）|
| `AWS_REGION` | 既存（DynamoDB / Cognito 共通）|



### Users テーブルの責務（変更）

- **Cognito が `userId / email / name / passwordHash / createdAt` を管理**
- DynamoDB の Users テーブルは **PoC では不要**（Cognito 一本化）
- Phase 2 構想：Sloth Feed 固有メタデータ（プロフィール画像 URL・好み設定など）が必要になれば DynamoDB Users テーブルを追加

Auth.js の薄い設定（`auth.ts`）が認証エントリポイント。

### Post.authorName の取得経路（更新）

- **API Route で `session.user.name` を直接取得 → PostService に渡す**
- UserRepository.findById は不要
- denormalization は維持（Post に authorName をスナップショット保存）

### トークン期限切れ・ログアウト

- Auth.js が **Cookie 期限と Cognito の RefreshToken** を組み合わせて自動管理
- ユーザーログアウト：`signOut()` ヘルパ → Auth.js が Cookie クリア + Cognito GlobalSignOut（任意）

### 未ログイン閲覧の許容

- `/` タイムライン・`/api/feed` は **middleware の matcher に含めない**（未ログイン閲覧可、Stage 3 で確定）
