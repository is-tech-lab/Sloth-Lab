# Unit of Work Dependency — Refactor the World (RTW) MVP

> このドキュメントは `component-dependency.md` を統合し、
> フィーチャーユニット単位で再整理したものです。

## フィーチャーユニット間依存関係マトリクス

| 依存元 → 依存先 | shared | Unit 1: Auth | Unit 2: Capture | Unit 3: Social Feed |
|---------------|:------:|:------------:|:---------------:|:-------------------:|
| **packages/shared** | — | — | — | — |
| **Unit 1: 認証** | ✓ 型定義 (build) | — | — | — |
| **Unit 2: Capture & Refactor** | ✓ 型定義 (build) | ✓ 実行時 (JWT検証) | — | — |
| **Unit 3: Social Feed** | ✓ 型定義 (build) | ✓ 実行時 (JWT検証) | ✓ データ (画像URLを文字列として受取) | — |
| **Unit 4: My Page** | ✓ 型定義 (build) | ✓ 実行時 (JWT検証) | — | ✓ コード (IPostRepository / ILikeRepositoryにメソッド追加) |

**凡例**:
- `型定義 (build)` = TypeScript型のimport。ビルド時のみの依存。実行時には影響しない
- `実行時 (JWT検証)` = JWTAuthMiddleware経由でリクエストを認証する。Unit 1のコードを直接importするのではなく、署名済みトークンを検証する
- `データ (画像URLを文字列として受取)` = Unit 2が生成したCDN画像URLをAPIのリクエストパラメータとして受け取るだけ。Unit 2のコードに依存しない
- `コード (Repository拡張)` = Unit 4はUnit 3で定義した `IPostRepository` / `ILikeRepository` に `findByUserId()` メソッドを追加する。定義の拡張依存がある

---

## 独立性の根拠

| ユニット | 他ユニットのコードへの依存 | 独立開発の根拠 |
|---------|--------------------------|--------------|
| **Unit 1: Auth** | なし | JWTを発行するだけ。他機能に依存しない |
| **Unit 2: Capture & Refactor** | Unit 1のJWT検証のみ（実行時） | カメラ・AI変換パイプラインはUnit 1完了後に独立実装できる |
| **Unit 3: Social Feed** | Unit 1のJWT検証のみ（実行時） | POST /postsは画像URLを文字列で受け取るだけ。Unit 2の実装は不要 |
| **Unit 4: My Page** | Unit 3のRepository interface拡張 | Unit 3のDBテーブルへの追加クエリ。Unit 3完了後に追加実装できる |

---

## コンポーネント内依存関係（DDDレイヤールール）

### Backend API — 依存方向

```
Presenter Layer (Express Routes)
    └──依存──→  Usecase Layer
                    └──依存──→  Repository Layer (Interfaces のみ)
                    └──依存──→  Domain Layer (Entities)
                    └──依存──→  IAIServiceClient (Interface)

Infrastructure (実装クラス — DI で注入)
    ├── PrismaUserRepository  implements IUserRepository
    ├── PrismaPostRepository  implements IPostRepository
    ├── PrismaLikeRepository  implements ILikeRepository
    ├── AWSS3Repository       implements IS3Repository
    └── HTTPAIServiceClient   implements IAIServiceClient
```

**ルール**: 上位レイヤーは下位レイヤーの**インターフェースにのみ**依存する。実装クラスへの直接依存は禁止。

### AI Integration Service — 依存方向

```
Presenter Layer (POST /transform)
    └──依存──→  Usecase Layer (ExecuteTransformPipelineUsecase)
                    └──依存──→  IOpenAIRepository (Interface)
                    └──依存──→  IS3StorageRepository (Interface)
                    └──依存──→  Domain Layer (TransformRequest entity)

Infrastructure (実装クラス — DI で注入)
    ├── OpenAIRepository       implements IOpenAIRepository
    └── AWSS3StorageRepository implements IS3StorageRepository
```

### Mobile App — 依存方向

```
Screens
    └──依存──→  Zustand Stores (読取・アクション呼出)
    └──依存──→  API Client Services (直接呼出は最小限)

Zustand Stores
    └──依存──→  API Service Classes (非同期アクション内)

API Service Classes
    └──依存──→  APIClient (Axiosインスタンス)

Navigation (RootNavigator)
    └──依存──→  AuthStore.isAuthenticated (ルート切り替え判定のみ)
```

**禁止**: Zustand StoreがScreensをimportしてはならない。

---

## 循環依存の禁止事項

| 禁止パターン | 理由 |
|------------|------|
| Usecase Layer → Presenter Layer | DDDレイヤー原則違反 |
| Domain Layer → Prisma / 外部ライブラリ | ドメインの汚染 |
| AI Service → Backend APIのRepository | AI ServiceはDBを持たない |
| Mobile Stores → Screens | 循環import |
| Unit 3 → Unit 2のコード直接import | フィーチャーユニット独立性違反 |

---

## フィーチャーユニット別データフロー

### Unit 1: 認証フロー

#### 新規登録 / ログイン

```
Mobile App                 Backend API              RDS
RegisterScreen
  │ 入力送信
  ▼
AuthAPIService
  .register()
  │ HTTPS POST /auth/register
  ▼
AuthRouter → RegisterUserUsecase
                 │
                 ├─→ IUserRepository.findByEmail()  →  [RDS: SELECT]
                 ├─→ PasswordHash.hash(password)
                 ├─→ IUserRepository.create()       →  [RDS: INSERT users]
                 └─→ JWTToken.generate()
                          │
                     AuthOutput { token, user }
  │ ◀──────────────────────────────────────────────
AuthStore.login(token, user)
RootNavigator → MainTabNavigator
```

#### ログアウト

```
Mobile App                 Backend API
SettingsScreen
  │ ログアウトタップ
  ▼
AuthAPIService (optional POST /auth/logout)
AuthStore.logout()
  │ token, currentUser をクリア
RootNavigator → AuthStack
```

---

### Unit 2: Capture & Refactorフロー

#### before画像アップロード

```
Mobile App                  Backend API              S3
CameraScreen
  │ 撮影 / カメラロール選択
  ▼
TransformStore.setBeforeImage(uri)
UploadAPIService
  .uploadImage(uri)
  │
  │ Step 1: HTTPS GET /upload/presigned-url
  ▼
UploadRouter → GeneratePresignedUrlUsecase
                    │
                    └─→ IS3Repository.generatePresignedUrl()  →  [S3: 署名付きURL生成]
                               { presignedUrl, cdnUrl }
  │ ◀───────────────────────────────────────────────────────────
  │
  │ Step 2: HTTPS PUT presignedUrl (S3へ直接アップロード)
  ▼
                                                        [S3: PUT image]
                                                              │ 200 OK
  │ ◀──────────────────────────────────────────────────────────
TransformStore.setBeforeImageUrl(cdnUrl)
→ TransformScreen遷移
```

#### AI変換実行

```
Mobile App             Backend API          AI Service           OpenAI / S3
TransformScreen
  │ Refactorボタンタップ
  ▼
TransformStore.setLoading(true)
TransformAPIService
  .requestTransform(beforeImageUrl)
  │ HTTPS POST /transform
  ▼
TransformRouter → RequestTransformUsecase
                       │ HTTP POST /transform (Internal ALB)
                       ▼
               TransformRequestRouter → ExecuteTransformPipelineUsecase
                                              │
                                              │ Step 1: analyzeImage
                                              ▼
                                       IOpenAIRepository → [GPT-4V API]
                                              │ ← prompt
                                              │ Step 2: generateImage
                                              ▼
                                       IOpenAIRepository → [DALL-E 3 API]
                                              │ ← imageBuffer
                                              │ Step 3: uploadImage
                                              ▼
                                  IS3StorageRepository → [S3: PUT after画像]
                                              │ ← cdnUrl
                              { afterImageUrl: cdnUrl }
                       │ ◀──────────────────────────────────────────
               { beforeImageUrl, afterImageUrl }
  │ ◀──────────────────────
TransformStore.setAfterImage(afterImageUrl)
TransformStore.setLoading(false)
→ before/after確認表示
```

---

### Unit 3: Social Feedフロー

#### 投稿

```
Mobile App                  Backend API              RDS
PostFormScreen
  │ カテゴリ選択 → 投稿ボタン
  ▼
PostAPIService
  .createPost({ beforeImageUrl, afterImageUrl, categoryTag })
  │ HTTPS POST /posts
  ▼
PostRouter → CreatePostUsecase
                 │
                 └─→ IPostRepository.create()  →  [RDS: INSERT posts]
                              │
                          PostOutput
  │ ◀────────────────────────────────────────────────
TransformStore.reset()
→ FeedScreen遷移
```

#### フィード閲覧

```
Mobile App                  Backend API              RDS / S3+CloudFront
FeedScreen (初回表示)
  │
  ▼
FeedStore.loadFeed()
FeedAPIService
  .getFeed(page=1)
  │ HTTPS GET /feed
  ▼
FeedRouter → GetFeedUsecase
                 │
                 └─→ IPostRepository.findFeed()  →  [RDS: SELECT posts JOIN users]
                              │
                          FeedOutput { posts[], hasNextPage }
  │ ◀────────────────────────────────────────────────
FeedStore.setPosts(posts)
画像表示: afterImageUrl → CloudFront CDN (直接)
```

#### いいね（楽観的更新）

```
Mobile App                  Backend API              RDS
FeedScreen
  │ いいねボタンタップ
  ▼
FeedStore.toggleLike(postId, true)   ← 楽観的更新（即時UI反映）
LikeAPIService.likePost(postId)
  │ HTTPS POST /likes/:postId
  ▼
LikeRouter → LikePostUsecase
                 ├─→ ILikeRepository.findByUserAndPost()  →  [RDS: SELECT]
                 ├─→ ILikeRepository.create()              →  [RDS: INSERT likes]
                 └─→ IPostRepository.incrementLikesCount() →  [RDS: UPDATE posts]
                         │ 200 OK / 409 Conflict (already liked)
  │ ◀──────────────────────────────────────────────────────────
失敗時: FeedStore.toggleLike(postId, false)  ← ロールバック
```

---

### Unit 4: My Pageフロー

#### 自分の投稿一覧 / いいねした投稿一覧

```
Mobile App                  Backend API              RDS
MyPageScreen
  │ タブ切り替え（投稿 / いいね）
  ▼
UserAPIService.getMyPosts(page)
  │ HTTPS GET /users/me/posts
  ▼
UserRouter → GetMyPostsUsecase
                 │
                 └─→ IPostRepository.findByUserId()  →  [RDS: SELECT posts WHERE userId=me]
                              │
                          FeedOutput
  │ ◀────────────────────────────────────────────────
MyPageScreen: 投稿グリッド表示

─── いいねした投稿 ───────────────────────────────────────────────────
UserAPIService.getLikedPosts(page)
  │ HTTPS GET /users/me/likes
  ▼
UserRouter → GetLikedPostsUsecase
                 │
                 └─→ ILikeRepository.findByUserId()   →  [RDS: SELECT likes JOIN posts WHERE userId=me]
                              │
                          FeedOutput
  │ ◀────────────────────────────────────────────────
MyPageScreen: いいね一覧表示
```

---

## ブロッキング依存（開発順序に影響）

```
packages/shared
    └─ ブロック ──→ Unit 1, 2, 3, 4（TypeScript型のimportに必要）

Unit 1: Auth
    └─ ブロック ──→ Unit 2, 3, 4（JWTAuthMiddleware が実装されないと認証が通らない）
    ※ 開発中はモック認証で代替可能（JWTAuthMiddlewareをスタブにする）

Unit 3: Social Feed (IPostRepository / ILikeRepository の定義)
    └─ ブロック ──→ Unit 4（findByUserId を追加するベース定義が必要）
```

**推奨開発順序**:
```
packages/shared → Unit 1 → Unit 2 → Unit 3 → Unit 4
```

Unit 2 と Unit 3 は Unit 1 完了後であれば並行開発可能（JWTAuthMiddlewareをスタブに置き換えれば Unit 1 完了前でも開発着手可能）。

---

## ユニット間インターフェース仕様（実行時依存）

### Backend API → AI Integration Service

```
POST http://ai-service-internal-alb/transform
Authorization: X-Internal-Secret: <shared-secret>
Content-Type: application/json

Request:  { "beforeImageUrl": "https://cdn.rtw.app/images/before-xxx.jpg" }
Response: { "afterImageUrl":  "https://cdn.rtw.app/images/after-xxx.jpg" }
Timeout:  10,000ms
```

### Mobile App → Backend API

```
Base URL: https://api.rtw.app (本番) / http://localhost:3000 (ローカル)
Authorization: Bearer <JWT token>
Content-Type: application/json
```

全エンドポイントのTypeScriptシグネチャは `unit-of-work.md` の各ユニットセクションを参照。
