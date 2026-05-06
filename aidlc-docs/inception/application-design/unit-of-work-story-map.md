# Unit of Work Story Map — Refactor the World (RTW) MVP

> このドキュメントはフィーチャーユニット定義への移行に合わせて再整理しました。
> 各ストーリーはいずれか1つのフィーチャーユニットに属します。

## ストーリー→ユニット対応表

| US ID | ストーリー | Priority | フィーチャーユニット |
|-------|----------|:--------:|:---------------:|
| US-01 | 新規ユーザー登録 | P1 | **Unit 1: 認証** |
| US-02 | ログイン | P1 | **Unit 1: 認証** |
| US-03 | ログアウト | P1 | **Unit 1: 認証** |
| US-04 | アプリ内カメラで撮影 | P0 | **Unit 2: Capture & Refactor** |
| US-05 | カメラロールから選択 | P0 | **Unit 2: Capture & Refactor** |
| US-06 | AI変換を実行 | P0 | **Unit 2: Capture & Refactor** |
| US-07 | 変換結果確認・再変換 | P0 | **Unit 2: Capture & Refactor** |
| US-08 | カテゴリタグ付きで投稿 | P1 | **Unit 3: Social Feed** |
| US-09 | フィードを閲覧 | P1 | **Unit 3: Social Feed** |
| US-10 | いいねをつける | P1 | **Unit 3: Social Feed** |
| US-11 | いいねを取り消す | P1 | **Unit 3: Social Feed** |
| US-12 | 自分の投稿一覧 | P2 | **Unit 4: My Page** |
| US-13 | いいねした投稿一覧 | P2 | **Unit 4: My Page** |

**全13ストーリーが割り当て済み ✓**

---

## Unit 1: 認証（3ストーリー）

### US-01: 新規ユーザー登録 [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /auth/register` (AuthRouter) |
| Backend: Usecase | `RegisterUserUsecase` |
| Backend: Repository | `IUserRepository.create()`, `PrismaUserRepository` |
| Backend: Domain | `User` entity, `PasswordHash`, `JWTToken` |
| Mobile: Screen | `RegisterScreen` |
| Mobile: Store | `AuthStore.login()` |
| Mobile: Service | `AuthAPIService.register()` |
| Mobile: Navigation | `AuthStack` → `MainTabNavigator` への遷移 |

**受け入れ基準の核心**: メール重複エラー（400）、パスワードバリデーション、JWT返却、自動ログイン状態遷移

---

### US-02: ログイン [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /auth/login` (AuthRouter) |
| Backend: Usecase | `LoginUserUsecase` |
| Backend: Repository | `IUserRepository.findByEmail()`, `PrismaUserRepository` |
| Backend: Domain | `PasswordHash.verify()`, `JWTToken.generate()` |
| Mobile: Screen | `LoginScreen` |
| Mobile: Store | `AuthStore.login()` |
| Mobile: Service | `AuthAPIService.login()` |

**受け入れ基準の核心**: 認証失敗時401、JWT返却、認証済み状態遷移

---

### US-03: ログアウト [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /auth/logout` (AuthRouter) ※オプション |
| Backend: Usecase | `LogoutUserUsecase` |
| Mobile: Screen | 設定画面内ログアウトボタン |
| Mobile: Store | `AuthStore.logout()` |
| Mobile: Navigation | `RootNavigator` → `AuthStack` への遷移 |

**受け入れ基準の核心**: ローカルJWTクリア、AuthStack復帰

---

## Unit 2: Capture & Refactor（4ストーリー）

### US-04: アプリ内カメラで撮影 [P0]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Mobile: Screen | `CameraScreen` (expo-camera) |
| Mobile: Store | `TransformStore.setBeforeImage(uri)` |
| Mobile: Navigation | CameraScreen → TransformScreen遷移 |

**受け入れ基準の核心**: カメラ権限リクエスト、シャッター操作、撮影後TransformScreen自動遷移

---

### US-05: カメラロールから選択 [P0]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Mobile: Screen | `CameraScreen` (expo-image-picker) |
| Mobile: Store | `TransformStore.setBeforeImage(uri)` |

**受け入れ基準の核心**: メディアライブラリ権限リクエスト、画像選択後TransformScreen遷移

---

### US-06: AI変換を実行 [P0]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `GET /upload/presigned-url` (UploadRouter), `POST /transform` (TransformRouter) |
| Backend: Usecase | `GeneratePresignedUrlUsecase`, `RequestTransformUsecase` |
| Backend: Repository | `IS3Repository`, `AWSS3Repository`, `IAIServiceClient`, `HTTPAIServiceClient` |
| AI Service: Presenter | `POST /transform` (TransformRequestRouter), `InternalAuthMiddleware` |
| AI Service: Usecase | `ExecuteTransformPipelineUsecase` (GPT-4V → DALL-E 3 → S3) |
| AI Service: Repository | `IOpenAIRepository`, `OpenAIRepository`, `IS3StorageRepository`, `AWSS3StorageRepository` |
| AI Service: Domain | `TransformRequest` entity, `TransformStatus` enum |
| Mobile: Screen | `TransformScreen` (ローディング表示) |
| Mobile: Store | `TransformStore.setLoading()`, `setAfterImage()` |
| Mobile: Service | `UploadAPIService.uploadImage()`, `TransformAPIService.requestTransform()` |

**受け入れ基準の核心**: S3直接アップロード、10秒以内のAI変換完了、after画像表示

---

### US-07: 変換結果確認・再変換 [P0]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /transform` (TransformRouter) — 再呼び出し |
| Mobile: Screen | `TransformScreen` (before/after並列表示, 再変換ボタン) |
| Mobile: Store | `TransformStore.reset()` → 再変換フロー |
| Mobile: Service | `TransformAPIService.requestTransform()` — 再呼び出し |

**受け入れ基準の核心**: before/after並列比較表示、再変換ボタン、変換失敗時のリトライ表示

---

## Unit 3: Social Feed（4ストーリー）

### US-08: カテゴリタグ付きで投稿 [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /posts` (PostRouter) |
| Backend: Usecase | `CreatePostUsecase` |
| Backend: Repository | `IPostRepository.create()`, `PrismaPostRepository` |
| Backend: Domain | `Post` entity |
| Mobile: Screen | `PostFormScreen` (カテゴリタグ選択) |
| Mobile: Store | `PostStore` (フォーム一時状態), `TransformStore.reset()` |
| Mobile: Service | `PostAPIService.createPost()` |

**受け入れ基準の核心**: カテゴリタグ必須（未選択でブロック）、投稿成功後FeedScreen遷移

---

### US-09: フィードを閲覧 [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `GET /feed` (FeedRouter) |
| Backend: Usecase | `GetFeedUsecase` |
| Backend: Repository | `IPostRepository.findFeed()`, `PrismaPostRepository` |
| Mobile: Screen | `FeedScreen` (スクロールリスト, ページネーション) |
| Mobile: Store | `FeedStore.setPosts()`, `appendPosts()` |
| Mobile: Service | `FeedAPIService.getFeed()` |
| AWS | S3 + CloudFront (after画像をCDN経由で表示) |

**受け入れ基準の核心**: 新着順表示、無限スクロール、初回表示3秒以内

---

### US-10: いいねをつける [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `POST /likes/:postId` (LikeRouter) |
| Backend: Usecase | `LikePostUsecase` |
| Backend: Repository | `ILikeRepository.create()`, `IPostRepository.incrementLikesCount()` |
| Mobile: Screen | `FeedScreen` (いいねボタン) |
| Mobile: Store | `FeedStore.toggleLike(postId, true)` (楽観的更新) |
| Mobile: Service | `LikeAPIService.likePost()` |

**受け入れ基準の核心**: 楽観的更新（タップ即時反映）、重複いいね防止（409）、失敗時ロールバック

---

### US-11: いいねを取り消す [P1]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `DELETE /likes/:postId` (LikeRouter) |
| Backend: Usecase | `UnlikePostUsecase` |
| Backend: Repository | `ILikeRepository.delete()`, `IPostRepository.decrementLikesCount()` |
| Mobile: Screen | `FeedScreen` (いいね取消) |
| Mobile: Store | `FeedStore.toggleLike(postId, false)` (楽観的更新) |
| Mobile: Service | `LikeAPIService.unlikePost()` |

**受け入れ基準の核心**: 楽観的更新、US-10と対称的な実装

---

## Unit 4: My Page（2ストーリー）

### US-12: 自分の投稿一覧 [P2]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `GET /users/me/posts` (UserRouter) |
| Backend: Usecase | `GetMyPostsUsecase` |
| Backend: Repository | `IPostRepository.findByUserId()` (Unit 3の定義に追加) |
| Mobile: Screen | `MyPageScreen` 「投稿」タブ |
| Mobile: Service | `UserAPIService.getMyPosts()` |

**受け入れ基準の核心**: 自分の投稿のみ表示、新着順、ページネーション

---

### US-13: いいねした投稿一覧 [P2]

| レイヤー | 実装コンポーネント |
|---------|-----------------|
| Backend: Presenter | `GET /users/me/likes` (UserRouter) |
| Backend: Usecase | `GetLikedPostsUsecase` |
| Backend: Repository | `ILikeRepository.findByUserId()` (Unit 3の定義に追加) |
| Mobile: Screen | `MyPageScreen` 「いいね」タブ |
| Mobile: Service | `UserAPIService.getLikedPosts()` |

**受け入れ基準の核心**: 自分がいいねした投稿のみ表示、新着いいね順

---

## ユニット別ストーリー数サマリー

| フィーチャーユニット | ストーリー数 | Priority 内訳 |
|-----------------|:----------:|:------------:|
| Unit 1: 認証 | 3 | P1 × 3 |
| Unit 2: Capture & Refactor | 4 | P0 × 4 |
| Unit 3: Social Feed | 4 | P1 × 4 |
| Unit 4: My Page | 2 | P2 × 2 |
| **合計** | **13** | P0×4 / P1×7 / P2×2 |

---

## 全ストーリー割当確認

- [x] US-01 — Unit 1: 認証
- [x] US-02 — Unit 1: 認証
- [x] US-03 — Unit 1: 認証
- [x] US-04 — Unit 2: Capture & Refactor
- [x] US-05 — Unit 2: Capture & Refactor
- [x] US-06 — Unit 2: Capture & Refactor
- [x] US-07 — Unit 2: Capture & Refactor
- [x] US-08 — Unit 3: Social Feed
- [x] US-09 — Unit 3: Social Feed
- [x] US-10 — Unit 3: Social Feed
- [x] US-11 — Unit 3: Social Feed
- [x] US-12 — Unit 4: My Page
- [x] US-13 — Unit 4: My Page

**全13ストーリーが割り当て済み ✓**
