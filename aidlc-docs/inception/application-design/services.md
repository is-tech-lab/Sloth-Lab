# Services — Refactor the World (RTW) MVP

## 概要

このドキュメントはサービス定義とオーケストレーションパターンを定義します。
RTWの主要フロー4本を中心に、コンポーネント間の協調を記述します。

---

## Service 1: ユーザー認証サービス（Auth Flow）

**担当ユニット**: Backend API

**フロー概要**: 新規登録・ログイン

```
[Mobile] → AuthAPIService.register/login
    → [Backend API] AuthRouter
    → RegisterUserUsecase / LoginUserUsecase
        → PrismaUserRepository.findByEmail (重複確認 / 認証)
        → PrismaUserRepository.create (新規登録のみ)
        → PasswordHash.hash / verify
        → JWTToken.generate
    ← AuthOutput { token, user }
[Mobile] → AuthStore.login(token, user)
```

**オーケストレーション責務**:
- `RegisterUserUsecase`: メール重複確認 → パスワードハッシュ化 → ユーザー作成 → JWT発行
- `LoginUserUsecase`: メール検索 → パスワード照合 → JWT発行

---

## Service 2: 画像アップロードサービス（Upload Flow）

**担当ユニット**: Backend API + Mobile + AWS S3

**フロー概要**: モバイルからS3へのPresigned URL直接アップロード

```
Step 1: Presigned URL取得
[Mobile] → UploadAPIService.getPresignedUrl(filename, contentType)
    → [Backend API] UploadRouter
    → GeneratePresignedUrlUsecase
        → AWSS3Repository.generatePresignedUrl(key, contentType)
    ← { presignedUrl, imageKey, cdnUrl }

Step 2: S3直接アップロード（バックエンドをバイパス）
[Mobile] → fetch(presignedUrl, PUT, imageData)
    → [AWS S3] 直接アップロード
    ← HTTP 200 OK

Result: Mobile が cdnUrl（CloudFront URL）を保持
```

**オーケストレーション責務**:
- `UploadAPIService`（Mobile）: 2ステップを連続実行するヘルパー `uploadImage()` を提供
- `GeneratePresignedUrlUsecase`: S3オブジェクトキーを生成し、Presigned URLとCDN URLを返す

**パフォーマンス目標**: 画像アップロード 5秒以内（Wi-Fi環境）

---

## Service 3: AI変換パイプラインサービス（Transform Flow）

**担当ユニット**: Backend API + AI Integration Service + AWS S3

**フロー概要**: before画像 → GPT-4V解析 → DALL-E 3生成 → after画像S3保存

```
[Mobile] → TransformAPIService.requestTransform(beforeImageUrl)
    → [Backend API] TransformRouter
    → RequestTransformUsecase
        → HTTPAIServiceClient.requestTransform(beforeImageUrl)
            → [AI Integration Service - Internal ALB]
            → ExecuteTransformPipelineUsecase
                Step 1: OpenAIRepository.analyzeImage(beforeImageUrl)
                        → GPT-4V API: 画像解析 → 改善方向プロンプト生成
                Step 2: OpenAIRepository.generateImage(prompt)
                        → DALL-E 3 API: プロンプト → after画像生成（Buffer）
                Step 3: AWSS3StorageRepository.uploadImage(buffer, key)
                        → S3: after画像保存
                        → CloudFront URL取得
                ← { afterImageUrl }
    ← { beforeImageUrl, afterImageUrl }
[Mobile] → TransformStore.setAfterImage(afterImageUrl)
         → TransformScreenで before/after 表示
```

**オーケストレーション責務**:
- `ExecuteTransformPipelineUsecase`: GPT-4V → DALL-E 3 → S3の3ステップを順次実行。失敗時にエラーを上位に伝播
- `RequestTransformUsecase`: AI Serviceへの委譲とタイムアウト管理（10秒目標）

**パフォーマンス目標**: AI変換レスポンス 10秒以内（OpenAI API応答速度に依存）

---

## Service 4: 投稿公開サービス（Post Publishing Flow）

**担当ユニット**: Backend API

**フロー概要**: before/afterペアをRTWフィードに投稿

```
[Mobile] → PostAPIService.createPost({ beforeImageUrl, afterImageUrl, categoryTag })
    → [Backend API] PostRouter
    → CreatePostUsecase
        → PrismaPostRepository.create(postData)
    ← PostOutput
[Mobile] → FeedStore.setPosts() or navigate to FeedScreen
```

**オーケストレーション責務**:
- `CreatePostUsecase`: 投稿レコードをDBに作成。before/afterURLはCloudFrontドメインを前提

---

## Service 5: フィード閲覧サービス（Feed Service）

**担当ユニット**: Backend API + Mobile

**フロー概要**: 新着順フィードのページネーション取得

```
[Mobile] FeedScreen初期表示
    → FeedAPIService.getFeed(page=1)
    → [Backend API] FeedRouter
    → GetFeedUsecase
        → PrismaPostRepository.findFeed({ page, limit: 20 })
    ← { posts, hasNextPage }
[Mobile] → FeedStore.setPosts(posts)

[Mobile] スクロールダウン（追加ロード）
    → FeedAPIService.getFeed(page=2)
    → ... (同上)
    ← FeedStore.appendPosts(newPosts)
```

**パフォーマンス目標**: フィード初回表示 3秒以内

---

## Service 6: いいねサービス（Like Service）

**担当ユニット**: Backend API + Mobile

**フロー概要**: いいね付与・取り消し（楽観的更新）

```
[Mobile] いいねボタンタップ
    → FeedStore.toggleLike(postId, true) ← 楽観的更新（即時UI反映）
    → LikeAPIService.likePost(postId)
    → [Backend API] LikeRouter
    → LikePostUsecase
        → PrismaLikeRepository.create({ userId, postId })
        → PrismaPostRepository.incrementLikesCount(postId)
    ← 200 OK
    // 失敗時: FeedStore.toggleLike(postId, false) で元に戻す
```

---

## サービス間通信まとめ

| 通信経路 | プロトコル | 認証方式 |
|---------|----------|---------|
| Mobile → Backend API | HTTPS (REST) | JWT Bearer Token |
| Mobile → S3 | HTTPS | Presigned URL（有効期限付き） |
| Backend API → AI Service | HTTP (内部ALB) | 共有シークレットキー（Internal Auth） |
| AI Service → OpenAI | HTTPS | OpenAI API Key（Secrets Manager） |
| AI Service → S3 | HTTPS (AWS SDK) | IAM Role |
| Backend API → RDS | TCP (Prisma接続) | DB接続文字列（Secrets Manager） |
