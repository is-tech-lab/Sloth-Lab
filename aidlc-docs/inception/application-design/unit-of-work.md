# Unit of Work — Refactor the World (RTW) MVP

> このドキュメントは `components.md` と `component-methods.md` を統合し、
> フィーチャーユニット単位で再整理したものです。

## フィーチャーユニット概要

| # | ユニット名 | ストーリー | Priority | 独立性の根拠 |
|---|-----------|-----------|----------|------------|
| 1 | **認証 (Auth)** | US-01〜03 | P1 | 他ユニットに依存しない基盤。JWTを発行するのみ |
| 2 | **Capture & Refactor** | US-04〜07 | P0 | カメラ・AI変換パイプライン。ソーシャル機能に依存しない |
| 3 | **Social Feed** | US-08〜11 | P1 | 投稿・フィード・いいね。画像URLを入力として受けるだけでUnit2コードに依存しない |
| 4 | **My Page** | US-12〜13 | P2 | DBへの読み取りクエリのみ。他ユニットのコードに依存しない |

**開発順序**: Unit 1 → Unit 2 → Unit 3 → Unit 4（依存関係の少ない順）  
**共通前提**: packages/shared の型定義を先に作成する

---

## コード整理戦略（モノレポ）

```
/  (Sloth-Lab ワークスペースルート)
├── apps/
│   ├── backend-api/          # Backend API（Unit 1〜4 のAPIを実装）
│   │   ├── src/
│   │   │   ├── domain/       # Domain Layer
│   │   │   ├── usecase/      # Usecase Layer
│   │   │   ├── repository/   # Repository Layer (interfaces + Prisma実装)
│   │   │   └── presenter/    # Presenter Layer (Express routers)
│   │   └── prisma/schema.prisma
│   │
│   ├── ai-service/           # AI Integration Service（Unit 2 専用）
│   │   └── src/
│   │       ├── domain/
│   │       ├── usecase/
│   │       ├── repository/
│   │       └── presenter/
│   │
│   └── mobile/               # Mobile App（Unit 1〜4 の画面を実装）
│       └── src/
│           ├── screens/
│           ├── stores/
│           ├── services/
│           └── navigation/
│
├── packages/
│   └── shared/               # 共有型定義（全ユニットが参照）
│       └── src/types/
│
├── infra/                    # AWS CDK（全ユニットのインフラ）
│   └── lib/
│
├── package.json
└── pnpm-workspace.yaml
```

---

## Unit 1: 認証 (Auth)

**ストーリー**: US-01（新規登録）、US-02（ログイン）、US-03（ログアウト）

### Backend API コンポーネント

#### Presenter Layer
| コンポーネント | 責務 |
|-------------|------|
| `AuthRouter` | POST /auth/register, POST /auth/login, POST /auth/logout |
| `JWTAuthMiddleware` | Authorization ヘッダーのJWT検証（全認証ルートで使用） |

#### Usecase Layer
| コンポーネント | 責務 |
|-------------|------|
| `RegisterUserUsecase` | メール重複確認 → パスワードハッシュ化 → ユーザー作成 → JWT発行 |
| `LoginUserUsecase` | メール検索 → パスワード照合 → JWT発行 |
| `LogoutUserUsecase` | セッション終了処理 |
| `GetMeUsecase` | 認証済みユーザーのプロフィール返却 |

#### Repository Layer
| コンポーネント | 責務 |
|-------------|------|
| `IUserRepository` | Userエンティティのデータアクセスインターフェース |
| `PrismaUserRepository` | `IUserRepository` のPrisma実装 |

#### Domain Layer
| コンポーネント | 責務 |
|-------------|------|
| `User` | ユーザーエンティティ（id, email, passwordHash, username, points: NULL） |
| `JWTToken` | JWT値オブジェクト。発行・検証ロジック |
| `PasswordHash` | パスワード値オブジェクト。bcryptハッシュ化・照合ロジック |

### Mobile App コンポーネント

| コンポーネント | 責務 |
|-------------|------|
| `LoginScreen` | メール・パスワード入力フォーム + ログインボタン |
| `RegisterScreen` | ユーザー名・メール・パスワード入力フォーム + 登録ボタン |
| `AuthStore` | JWT token, currentUser のグローバル状態。login/logout アクション |
| `AuthAPIService` | register, login の APIリクエストラッパー |
| `AuthStack` | LoginScreen + RegisterScreen のスタックナビゲーション |

### AWS コンポーネント（Auth）

| コンポーネント | 責務 |
|-------------|------|
| Secrets Manager（JWT秘密鍵） | JWT署名用シークレットの安全な管理 |
| RDS `users` テーブル | ユーザーデータの永続化 |

### キーメソッドシグネチャ（Unit 1）

```typescript
// --- Usecase Layer ---

interface RegisterUserInput { email: string; password: string; username: string }
interface AuthOutput { token: string; user: { id: string; email: string; username: string } }
class RegisterUserUsecase { execute(input: RegisterUserInput): Promise<AuthOutput> }

interface LoginUserInput { email: string; password: string }
class LoginUserUsecase  { execute(input: LoginUserInput):    Promise<AuthOutput> }
class LogoutUserUsecase { execute(userId: string):           Promise<void> }
class GetMeUsecase      { execute(userId: string):           Promise<UserOutput> }

// --- Repository Interface ---

interface IUserRepository {
  findById(id: string):       Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  create(data: { email: string; passwordHash: string; username: string }): Promise<User>
}

// --- Mobile Store ---

interface AuthState  { token: string | null; currentUser: UserData | null; isAuthenticated: boolean }
interface AuthActions { login(token: string, user: UserData): void; logout(): void }

// --- Mobile Service ---

class AuthAPIService {
  register(input: { email: string; password: string; username: string }): Promise<AuthOutput>
  login(input: { email: string; password: string }): Promise<AuthOutput>
}
```

---

## Unit 2: Capture & Refactor（カメラ + AI変換）

**ストーリー**: US-04（カメラ撮影）、US-05（カメラロール選択）、US-06（AI変換実行）、US-07（変換確認・再変換）

### Backend API コンポーネント

#### Presenter Layer
| コンポーネント | 責務 |
|-------------|------|
| `UploadRouter` | GET /upload/presigned-url（S3 Presigned URL発行） |
| `TransformRouter` | POST /transform（AI Serviceへの委譲エンドポイント） |

#### Usecase Layer
| コンポーネント | 責務 |
|-------------|------|
| `GeneratePresignedUrlUsecase` | S3オブジェクトキー生成 → Presigned URLとCDN URLを返す |
| `RequestTransformUsecase` | AI Integration Serviceにリクエスト委譲 → 結果をMobileに返す |

#### Repository Layer
| コンポーネント | 責務 |
|-------------|------|
| `IS3Repository` | S3操作インターフェース |
| `AWSS3Repository` | `IS3Repository` のAWS SDK実装 |
| `IAIServiceClient` | AI Serviceとの通信インターフェース |
| `HTTPAIServiceClient` | `IAIServiceClient` のHTTP実装 |

### AI Integration Service コンポーネント（Unit 2 専用サービス）

#### Presenter Layer
| コンポーネント | 責務 |
|-------------|------|
| `TransformRequestRouter` | POST /transform（Backend APIからの内部リクエストのみ受付） |
| `InternalAuthMiddleware` | 共有シークレットキーによる内部サービス認証 |

#### Usecase Layer
| コンポーネント | 責務 |
|-------------|------|
| `ExecuteTransformPipelineUsecase` | GPT-4V解析 → DALL-E 3生成 → S3アップロードの3ステップパイプライン |

#### Repository Layer
| コンポーネント | 責務 |
|-------------|------|
| `IOpenAIRepository` | OpenAI API操作インターフェース |
| `OpenAIRepository` | GPT-4V（画像解析）+ DALL-E 3（画像生成）のOpenAI SDK実装 |
| `IS3StorageRepository` | AI Service用S3操作インターフェース |
| `AWSS3StorageRepository` | after画像のS3アップロードを担当。CloudFront URLを返す |

#### Domain Layer
| コンポーネント | 責務 |
|-------------|------|
| `TransformRequest` | 変換リクエストエンティティ（beforeImageUrl, analysisResult, afterImageUrl, status） |
| `TransformStatus` | 変換ステータス列挙型（PENDING / ANALYZING / GENERATING / UPLOADING / COMPLETED / FAILED） |

### Mobile App コンポーネント

| コンポーネント | 責務 |
|-------------|------|
| `CameraScreen` | カメラ起動・シャッター操作・カメラロール選択 |
| `TransformScreen` | AI変換実行・ローディング表示・before/after確認・再変換ボタン |
| `TransformStore` | beforeImageUri, afterImageUrl, isLoading, error の状態管理 |
| `UploadAPIService` | Presigned URL取得 → S3直接アップロードの2ステップを管理 |
| `TransformAPIService` | POST /transform の APIリクエストラッパー |

### AWS コンポーネント（Capture & Refactor）

| コンポーネント | 責務 |
|-------------|------|
| S3 Bucket | before/after画像の保存 |
| CloudFront Distribution | 画像CDN配信（全画像URLはCloudFrontドメイン） |
| AI Service ECS Fargate | AI Integrationコンテナ実行 |
| ALB（Internal） | Backend API → AI Service の内部通信 |

### キーメソッドシグネチャ（Unit 2）

```typescript
// --- Backend API Usecase ---

interface PresignedUrlInput  { filename: string; contentType: string }
interface PresignedUrlOutput { presignedUrl: string; imageKey: string; cdnUrl: string }
class GeneratePresignedUrlUsecase { execute(input: PresignedUrlInput): Promise<PresignedUrlOutput> }

interface TransformInput  { beforeImageUrl: string }
interface TransformOutput { beforeImageUrl: string; afterImageUrl: string }
class RequestTransformUsecase { execute(input: TransformInput): Promise<TransformOutput> }

// --- Backend API Repository Interfaces ---

interface IS3Repository {
  generatePresignedUrl(key: string, contentType: string): Promise<{ presignedUrl: string; cdnUrl: string }>
}
interface IAIServiceClient {
  requestTransform(beforeImageUrl: string): Promise<{ afterImageUrl: string }>
}

// --- AI Service Usecase ---

interface TransformPipelineInput  { beforeImageUrl: string }
interface TransformPipelineOutput { afterImageUrl: string }
class ExecuteTransformPipelineUsecase {
  // Step 1: OpenAIRepository.analyzeImage → prompt
  // Step 2: OpenAIRepository.generateImage → imageBuffer
  // Step 3: AWSS3StorageRepository.uploadImage → cdnUrl
  execute(input: TransformPipelineInput): Promise<TransformPipelineOutput>
}

// --- AI Service Repository Interfaces ---

interface IOpenAIRepository {
  analyzeImage(imageUrl: string):   Promise<{ prompt: string }>
  generateImage(prompt: string):    Promise<{ imageBuffer: Buffer }>
}
interface IS3StorageRepository {
  uploadImage(buffer: Buffer, key: string, contentType: string): Promise<{ cdnUrl: string }>
}

// --- Mobile Store ---

interface TransformState {
  beforeImageUri: string | null; beforeImageUrl: string | null
  afterImageUrl: string | null; isLoading: boolean; error: string | null
}
interface TransformActions {
  setBeforeImage(uri: string): void; setBeforeImageUrl(url: string): void
  setAfterImage(url: string): void; setLoading(loading: boolean): void
  setError(error: string | null): void; reset(): void
}

// --- Mobile Services ---

class UploadAPIService {
  getPresignedUrl(filename: string, contentType: string): Promise<{ presignedUrl: string; cdnUrl: string }>
  uploadToS3(presignedUrl: string, imageData: Blob, contentType: string): Promise<void>
  uploadImage(imageUri: string): Promise<{ cdnUrl: string }>   // 上記2ステップの連続実行ヘルパー
}
class TransformAPIService {
  requestTransform(beforeImageUrl: string): Promise<TransformOutput>
}
```

---

## Unit 3: Social Feed（投稿 + フィード + いいね）

**ストーリー**: US-08（投稿）、US-09（フィード閲覧）、US-10（いいね）、US-11（いいね取消）

### Backend API コンポーネント

#### Presenter Layer
| コンポーネント | 責務 |
|-------------|------|
| `PostRouter` | POST /posts |
| `FeedRouter` | GET /feed |
| `LikeRouter` | POST /likes/:postId, DELETE /likes/:postId |

#### Usecase Layer
| コンポーネント | 責務 |
|-------------|------|
| `CreatePostUsecase` | before/after画像URLとカテゴリタグで投稿レコードを作成 |
| `GetFeedUsecase` | 新着順フィードをページネーション付きで返す |
| `LikePostUsecase` | いいねを記録し likes_count をインクリメント |
| `UnlikePostUsecase` | いいねを取り消し likes_count をデクリメント |

#### Repository Layer
| コンポーネント | 責務 |
|-------------|------|
| `IPostRepository` | Postエンティティのデータアクセスインターフェース |
| `PrismaPostRepository` | `IPostRepository` のPrisma実装 |
| `ILikeRepository` | Likeエンティティのデータアクセスインターフェース |
| `PrismaLikeRepository` | `ILikeRepository` のPrisma実装 |

#### Domain Layer
| コンポーネント | 責務 |
|-------------|------|
| `Post` | 投稿エンティティ（id, userId, beforeImageUrl, afterImageUrl, categoryTag, likesCount） |
| `Like` | いいねエンティティ（id, userId, postId）。UNIQUE(userId, postId) |

### Mobile App コンポーネント

| コンポーネント | 責務 |
|-------------|------|
| `PostFormScreen` | カテゴリタグ選択 + 投稿ボタン |
| `FeedScreen` | 他ユーザー投稿のスクロールリスト + いいねボタン |
| `FeedStore` | feedPosts一覧・ページネーション位置・いいね楽観的更新 |
| `PostStore` | 投稿フォームの一時状態（カテゴリタグ選択） |
| `PostAPIService` | POST /posts のリクエストラッパー |
| `FeedAPIService` | GET /feed（ページネーション対応）のリクエストラッパー |
| `LikeAPIService` | POST/DELETE /likes/:postId のリクエストラッパー |

### キーメソッドシグネチャ（Unit 3）

```typescript
// --- Usecase Layer ---

interface PostOutput { id: string; userId: string; beforeImageUrl: string; afterImageUrl: string; categoryTag: string; likesCount: number; createdAt: Date; user: { username: string } }
interface CreatePostInput { userId: string; beforeImageUrl: string; afterImageUrl: string; categoryTag: 'food' | 'city' | 'interior' | 'other' }
class CreatePostUsecase { execute(input: CreatePostInput): Promise<PostOutput> }

interface FeedOutput { posts: PostOutput[]; totalCount: number; hasNextPage: boolean }
class GetFeedUsecase   { execute(input: { page: number; limit: number }): Promise<FeedOutput> }
class LikePostUsecase  { execute(input: { userId: string; postId: string }): Promise<void> }
class UnlikePostUsecase{ execute(input: { userId: string; postId: string }): Promise<void> }

// --- Repository Interfaces ---

interface IPostRepository {
  create(data: { userId: string; beforeImageUrl: string; afterImageUrl: string; categoryTag: string }): Promise<Post>
  findFeed(input: { page: number; limit: number }): Promise<{ posts: Post[]; totalCount: number }>
  incrementLikesCount(postId: string): Promise<void>
  decrementLikesCount(postId: string): Promise<void>
}
interface ILikeRepository {
  findByUserAndPost(userId: string, postId: string): Promise<Like | null>
  create(data: { userId: string; postId: string }): Promise<Like>
  delete(userId: string, postId: string): Promise<void>
}

// --- Mobile Store ---

interface FeedState { posts: PostData[]; currentPage: number; hasNextPage: boolean; isLoading: boolean }
interface FeedActions {
  setPosts(posts: PostData[]): void
  appendPosts(posts: PostData[]): void
  toggleLike(postId: string, liked: boolean): void  // 楽観的更新
  setLoading(loading: boolean): void; reset(): void
}

// --- Mobile Services ---

class PostAPIService { createPost(input: { beforeImageUrl: string; afterImageUrl: string; categoryTag: string }): Promise<PostOutput> }
class FeedAPIService { getFeed(page: number, limit?: number): Promise<{ posts: PostData[]; hasNextPage: boolean }> }
class LikeAPIService { likePost(postId: string): Promise<void>; unlikePost(postId: string): Promise<void> }
```

---

## Unit 4: My Page（マイページ）

**ストーリー**: US-12（自分の投稿一覧）、US-13（いいねした投稿一覧）

### Backend API コンポーネント

#### Presenter Layer
| コンポーネント | 責務 |
|-------------|------|
| `UserRouter` | GET /users/me, GET /users/me/posts, GET /users/me/likes |

#### Usecase Layer
| コンポーネント | 責務 |
|-------------|------|
| `GetMyPostsUsecase` | 認証済みユーザーの投稿一覧をページネーション付きで返す |
| `GetLikedPostsUsecase` | 認証済みユーザーがいいねした投稿一覧を返す |

#### Repository Layer（既存の再利用）
| コンポーネント | 再利用元 |
|-------------|---------|
| `IPostRepository.findByUserId()` | Unit 3で定義 |
| `ILikeRepository.findByUserId()` | Unit 3で定義 |

### Mobile App コンポーネント

| コンポーネント | 責務 |
|-------------|------|
| `MyPageScreen` | 「投稿」「いいね」タブ切り替え。PostListを表示 |
| `UserAPIService` | getMyPosts, getLikedPosts の APIリクエストラッパー |

### キーメソッドシグネチャ（Unit 4）

```typescript
// --- Usecase Layer ---

class GetMyPostsUsecase    { execute(input: { userId: string; page: number; limit: number }): Promise<FeedOutput> }
class GetLikedPostsUsecase { execute(input: { userId: string; page: number; limit: number }): Promise<FeedOutput> }

// --- Repository Interface 追加メソッド ---

interface IPostRepository {
  // Unit 3 の定義に追加:
  findByUserId(userId: string, pagination: { page: number; limit: number }): Promise<{ posts: Post[]; totalCount: number }>
}
interface ILikeRepository {
  // Unit 3 の定義に追加:
  findByUserId(userId: string, pagination: { page: number; limit: number }): Promise<Like[]>
}

// --- Mobile Service ---

class UserAPIService {
  getMe(): Promise<UserData>
  getMyPosts(page: number): Promise<{ posts: PostData[]; hasNextPage: boolean }>
  getLikedPosts(page: number): Promise<{ posts: PostData[]; hasNextPage: boolean }>
}
```

---

## 共通コンポーネント（クロスカッティング）

全ユニットで横断的に使用されるコンポーネント。

### packages/shared（型定義）

```typescript
// 全ユニットが参照する共通型（apps/* が @rtw/shared としてimport）
export interface UserData      { id: string; email: string; username: string; createdAt: Date }
export interface PostData      { id: string; userId: string; beforeImageUrl: string; afterImageUrl: string; categoryTag: string; likesCount: number; createdAt: Date; user: { username: string } }
export interface AuthOutput    { token: string; user: UserData }
export interface FeedOutput    { posts: PostData[]; totalCount: number; hasNextPage: boolean }
export interface TransformOutput { beforeImageUrl: string; afterImageUrl: string }
```

### Mobile App 共通コンポーネント

| コンポーネント | 責務 |
|-------------|------|
| `APIClient` | Axiosインスタンス。JWTをインターセプターで自動付与。401時にAuthStoreをクリアしてログイン画面へ |
| `RootNavigator` | isAuthenticated に応じてAuthStack↔MainTabNavigatorを切り替え |
| `MainTabNavigator` | Feed / Camera / MyPage のボトムタブ |

### AWS Infrastructure（全ユニット共通）

| コンポーネント | 関連ユニット |
|-------------|------------|
| ECS Fargate（Backend API） | Unit 1〜4 すべて |
| ECS Fargate（AI Service） | Unit 2 のみ |
| RDS PostgreSQL | Unit 1（users）, Unit 3（posts/likes）, Unit 4（posts/likes読取） |
| S3 + CloudFront | Unit 2（画像保存）, Unit 3（画像表示） |
| ALB Public | Unit 1〜4（Mobile → Backend API） |
| ALB Internal | Unit 2（Backend API → AI Service） |
| Secrets Manager | Unit 1（JWT）, Unit 2（OpenAI API Key, Internal Secret）, 全ユニット（DB接続文字列） |
