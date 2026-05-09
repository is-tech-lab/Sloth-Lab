# コンポーネント依存関係 — Sloth Feed

> 最新版：2026-05-09 / 改訂履歴は [`audit.md`](../../audit.md) と [`aidlc-state.md`](../../aidlc-state.md) を参照。

## 依存関係マトリクス

| コンポーネント | auth.ts (Auth.js) | PostService | FeedService | AIFilteringService | AINamakemonoService | PostRepository | UserHistory | Bedrock | Cognito | DynamoDB |
|-------------|:------------:|:-----------:|:-----------:|:-----------------:|:----------------:|:--------------:|:-----------:|:-------:|:-------:|:--------:|
| API Route (`[...nextauth]`) | ○ | | | | | | | | | |
| API Route (posts) | ○ (auth() 呼び出し) | ○ | | | | | | | | |
| API Route (feed/my-posts) | ○ (auth() 呼び出し) | | ○ | | | | | | | |
| auth.ts (Auth.js) | — | | | | | | | | ○ | |
| PostService | | | | ○ | ○ | ○ | | | | |
| FeedService | | | | | | ○ | | | | |
| AIFilteringService | | | | | | | | ○ | | |
| AINamakemonoService | | | | | | | ○ | ○ | | |
| PostRepository | | | | | | | | | | ○ |
| UserHistory | | | | | | ○ | | | | ○ |
| middleware.ts | ○ (auth を default export) | | | | | | | | | |

---

## データフロー図

### フロー 1: 投稿作成（コアフロー）

```
クライアント
  │  POST /api/posts  { content }
  │  Cookie: Auth.js セッション Cookie（HttpOnly、自動送信）
  ↓
middleware.ts
  │  Auth.js の auth helper が Cookie を検証
  │  matcher 設定により /api/posts は保護対象、未認証は 401
  ↓
API Route: /api/posts/route.ts
  │  const session = await auth();  // Auth.js から取得
  │  authorId = session.user.id;    // Cognito sub
  │  authorName = session.user.name; // Cognito custom:name
  │  入力検証（content 1〜500 文字）
  ↓
PostService.createPost(authorId, authorName, content)
  │
  ├──→ AIFilteringService.filterPost(content)
  │         │  Bedrock Claude 呼び出し（怠惰系・善行系両方を通過）
  │         └─ FilterResult
  │              ├─ allowed: false
  │              │     └── { success: false, failureType: 'filtering_excluded', message }
  │              │              → API Route が 422 を返す
  │              └─ allowed: true
  │                    ↓
  │  （authorName は session.user.name から既に取得済み・UserRepository 呼び出しなし）
  │
  ├──→ AINamakemonoService.generateResponse(authorId, content)
  │         │
  │         ├──→ UserHistory.getRecent(authorId, N)
  │         │         └─ 過去投稿（個別化記憶 / FR-006）
  │         ├──→ UserHistory.getActivityMetrics(authorId)
  │         │         └─ 連続投稿数・滞在時間（依存防止 / FR-009）
  │         ├──→ プロンプト構築（5経路ヒント・老師人格 SystemPrompt・引用源マッピング）
  │         └──→ Bedrock 経由で Claude モデル呼び出し（IAM 認証）
  │              └─ NamakemonoResponse
  │                  { pathway, pathwayLabel, comment, citationSource, shouldSuggestBreak }
  │              ※ 失敗時 → { success: false, failureType: 'ai_generation_failed' }
  │
  └──→ PostRepository.create({
            content, authorId, authorName,
            aiComment, aiCitationSource, pathway
        })
            │  DynamoDB Posts テーブルに書き込み
            └─ Post
                 ※ 失敗時 → { success: false, failureType: 'persistence_failed' }
                 → API Route が 201 を返す
```

### フロー 2: ユーザー登録（Auth.js + Cognito OAuth/OIDC）

```
クライアント
  │  signIn("cognito") を呼び出し（自前フォームの場合）または
  │  Cognito Hosted UI へリダイレクト（PoC 実装時に決定）
  ↓
Auth.js が Cognito の OAuth/OIDC フローを開始
  │  /api/auth/[...nextauth] が Auth.js handlers を呼ぶ
  ↓
AWS Cognito User Pool
  │  email + password + custom:name を受け付け（または Hosted UI で入力）
  │  ユーザー作成 + パスワードハッシュ + email 検証フロー
  │  ID トークン + Access トークン + Refresh トークンを発行
  ↓
Auth.js のコールバック
  │  - JWT 検証（JWKS 自動取得）
  │  - Session callback で session.user.id = sub, session.user.name = custom:name
  │  - HttpOnly Cookie にセッション書き込み
  ↓
クライアント
  │  Cookie が自動セット → useSession() で { id, name } を取得可能
  │  タイムライン画面へリダイレクト
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
  │  DynamoDB Posts テーブルを Scan
  │  → アプリケーション層で createdAt 降順ソート（PoC 許容）
  └─ PaginatedResult<Post>
       → API Route が 200 を返す
```

**注意**: DynamoDB の Scan 操作は**ソート順を保証しない**。PoC では「全件 Scan + アプリ層で createdAt 降順ソート」で対応する（投稿数が増えると Scan コストが線形増加するが、PoC の制約として許容）。**Phase 2 構想**：`createdAt-index` GSI（PK は固定値・SK は createdAt）の追加検討。

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
| **Auth.js セッション** | middleware.ts (Auth.js auth) → API Route (`await auth()`) | **Cookie ベースのセッションを Auth.js が管理**。API Route は `await auth()` で session を取得し `session.user.id` / `session.user.name` を利用 |
| **OAuth/OIDC** | Auth.js → Cognito User Pool | **Cognito を OAuth プロバイダとして利用**。Auth.js が JWKS 検証・トークン管理を担う |
| AWS SDK 呼び出し | AIFilteringService / AINamakemonoService → Amazon Bedrock | `@aws-sdk/client-bedrock-runtime` 経由の Claude モデル呼び出し（IAM 認証）|
| **個別化記憶** | AINamakemonoService → UserHistory → PostRepository | **過去投稿と活動メトリクスを参照**して FR-006（記憶）・FR-009（依存防止判定）を実現 |
| **denormalization** | API Route (`session.user.name`) → PostService → PostRepository | **投稿時に authorName をスナップショット保存**（Cognito の name を Post に固定、読み取り高速化、整合性課題は機能設計で）|

---

## DynamoDB テーブル設計

### Posts テーブル

| キー | 型 | 説明 |
|------|----|------|
| postId (PK) | String | UUID v4 |
| content | String | 投稿本文 |
| authorId | String | 投稿者の userId |
| authorName | String | 投稿時の User.name をスナップショット（denormalization）|
| aiComment | String | AI生成の称賛コメント |
| aiCitationSource | String | 引用元（PoC では LLM 自己申告）|
| pathway | Number (1-5) | 紐付けられた経路（FR-011）|
| createdAt | String | ISO 8601（ソート用） |

- **GSI**: `authorId-createdAt-index`（PK: authorId, SK: createdAt）— getUserPosts に使用
- タイムライン（全件 createdAt 降順）は **Scan + アプリ層ソート**（PoC 許容）。**DynamoDB Scan はソート順を保証しないため、`PostRepository.findAll` 内でアプリケーション層が createdAt 降順にソートして返す**
- **Phase 2 構想**：`createdAt-index` GSI（PK は固定値 `'TIMELINE'` 等、SK は createdAt）を追加して O(log n) のソート済み取得に切り替え検討
