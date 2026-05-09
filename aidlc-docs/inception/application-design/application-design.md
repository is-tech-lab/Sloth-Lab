# アプリケーション設計 — Sloth Feed PoC（IP事業 × 動的IP × AI技術）

> **本ドキュメントの前提（2026-05-09 更新）**
> Issue #5 帰着により、Sloth Feed は IP事業として位置づけ直された。
> **技術選定（Next.js / Claude API / DynamoDB / JWT）は維持**しつつ、各ユニット・コンポーネントの**責務と意味を「動的IP × AI技術」文脈で再記述**している。構造変更はない。

---

## 設計サマリー

| 項目 | 決定内容 |
|------|---------|
| フレームワーク | Next.js 14+ App Router / TypeScript |
| サービスレイヤー | `lib/services/` に分離（薄いコントローラ + サービスクラス）|
| DBアクセス | リポジトリパターン（`lib/repositories/`）|
| DynamoDB | テーブル分割（Users / Posts）|
| AI サービス | **AIFilteringService**（仕事系投稿の弾き）と **AINamakemonoService**（旧 AICommentService、動的IPの対話）を分離 |
| RAG引用ライブラリ | **新規**：偉人・哲学・科学の引用源を管理する内部データベース／JSONファイル |
| 個別化記憶 | **新規**：ユーザー履歴を AI に渡すコンテキスト構築 |
| JWT 認証 | `middleware.ts` で一括検証 |
| UIコンポーネント | `app/`（ページ）+ `components/`（再利用 UI、**サンドイッチUI構造**）|

---

## ディレクトリ構造

```
sloth-feed/
├── app/
│   ├── (main)/
│   │   ├── page.tsx                   # タイムライン（サンドイッチUI）
│   │   ├── post/
│   │   │   └── page.tsx               # ダメ投稿フォーム
│   │   └── my-posts/
│   │       └── page.tsx               # 自分のダメ投稿一覧
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
│       │   └── route.ts               # POST: ダメ投稿作成
│       ├── feed/
│       │   └── route.ts               # GET: タイムライン
│       └── my-posts/
│           └── route.ts               # GET: 自分のダメ投稿一覧
├── components/
│   ├── PostCard.tsx                   # サンドイッチUIで投稿+AIコメントを表示
│   ├── PostForm.tsx                   # ダメ投稿フォーム
│   ├── FeedList.tsx                   # PostCard のリスト
│   ├── NamakemonoBubble.tsx           # AIナマケモノコメントの吹き出し（旧 AICommentBubble）
│   ├── BrandFrame.tsx                 # 「仕事じゃないけど…世の中を変える」サンドイッチUIの上下フレーム
│   ├── FilteringFeedback.tsx          # 仕事系投稿除外時のフィードバック
│   ├── AuthForm.tsx
│   └── LoadingSpinner.tsx
├── lib/
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── post.service.ts
│   │   ├── feed.service.ts
│   │   ├── ai-filtering.service.ts    # 仕事系投稿の弾き
│   │   └── ai-namakemono.service.ts   # 旧 ai-comment.service.ts。動的IPの対話エンジン
│   ├── repositories/
│   │   ├── user.repository.ts
│   │   └── post.repository.ts
│   ├── rag/                           # 新規：RAG引用ライブラリ
│   │   ├── citations.json             # 偉人引用データベース
│   │   └── retriever.ts               # 引用検索ロジック
│   ├── memory/                        # 新規：個別化記憶
│   │   └── user-history.ts            # ユーザー履歴の取得・コンテキスト構築
│   └── types/
│       └── index.ts
└── middleware.ts
```

---

## コンポーネント一覧

### バックエンド・サービス

| コンポーネント | パス | 主な責務（IP × 動的IP × AI 観点で再記述）|
|--------------|------|---------|
| AuthService | `lib/services/auth.service.ts` | 登録・ログイン・JWT発行。**IP のファン識別基盤** |
| PostService | `lib/services/post.service.ts` | ダメ投稿フローのオーケストレーション（フィルタリング → ナマケモノ対話 → 保存）|
| FeedService | `lib/services/feed.service.ts` | タイムライン・自分のダメ投稿一覧取得。**ファン共同体タイムライン** |
| **AIFilteringService** | `lib/services/ai-filtering.service.ts` | Claude API で仕事系投稿を弾く。**IPコンセプトの境界を守る** |
| **AINamakemonoService**（旧 AICommentService）| `lib/services/ai-namakemono.service.ts` | **動的IP の核**。Claude API + RAG引用ライブラリ + 個別化記憶で、ナマケモノとして個別化された肯定コメントを生成 |
| UserRepository | `lib/repositories/user.repository.ts` | Users テーブル CRUD |
| PostRepository | `lib/repositories/post.repository.ts` | Posts テーブル CRUD + GSI 検索 |
| **RAGRetriever**（新規）| `lib/rag/retriever.ts` | 引用ライブラリから投稿内容に応じた偉人引用を検索 |
| **UserHistory**（新規）| `lib/memory/user-history.ts` | ユーザーの過去投稿を取得し、AI コンテキストに渡す |
| middleware.ts | `middleware.ts` | 保護ルートの JWT 一括検証 |

### フロントエンド・UIコンポーネント

| コンポーネント | 説明 |
|--------------|------|
| **PostCard** | 投稿本文 + AI ナマケモノコメントを**サンドイッチUI構造**で表示。`BrandFrame` 内側に内容が挟まる |
| **PostForm** | ダメ投稿入力フォーム（仕事じゃないけど prefix 強制なし）|
| **FeedList** | PostCard のリスト |
| **NamakemonoBubble** | AI ナマケモノコメントの吹き出し（**引用元を明記**）|
| **BrandFrame**（新規）| サンドイッチUI の上下フレーム。「仕事じゃないけど…世の中を変える」を保証 |
| FilteringFeedback | 「ここはダメを誇る場所です」メッセージ表示 |
| AuthForm | ログイン・登録の共通フォーム |
| LoadingSpinner | Claude API 呼び出し中のローディング |

---

## コアフロー：ダメ投稿作成（動的IP対話）

```
クライアント → middleware.ts（JWT検証）
             → POST /api/posts
             → PostService.createPost
                  → AIFilteringService.filterPost（Claude API）
                       ├─ 除外: 422 + reason を返す
                       └─ 通過:
                  → AINamakemonoService.generateResponse
                       ├─ UserHistory.getRecent（個別化記憶）
                       ├─ RAGRetriever.search（引用検索）
                       └─ Claude API 呼び出し（ナマケモノ人格）
                  → PostRepository.create（DynamoDB）
                  → 201 + Post（aiComment + aiCitationSource 含む）を返す
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
| **aiCitationSource**（新規）| String | 引用元（例：「Larry Wall」「ラッセル『怠惰への讃歌』」）|
| createdAt | String (ISO) | GSI SK |

**変更点**：
- 旧 `stamps` フィールドを**削除**（スタンプ機能は廃止のため）
- `aiCitationSource` を**追加**（出典明記・ハルシネーション対策）

---

## API エンドポイント一覧

| メソッド | パス | 認証 | 説明 |
|---------|------|------|------|
| POST | /api/auth/register | なし | ユーザー登録 |
| POST | /api/auth/login | なし | ログイン・JWT発行 |
| POST | /api/posts | JWT | ダメ投稿作成（フィルタリング〜ナマケモノ応答）|
| GET | /api/feed | なし | タイムライン取得（未ログインでも閲覧可）|
| GET | /api/my-posts | JWT | 自分のダメ投稿一覧取得 |

---

## 環境変数

| 変数名 | 用途 |
|--------|------|
| ANTHROPIC_API_KEY | Claude API 認証 |
| DYNAMODB_USERS_TABLE | Users テーブル名 |
| DYNAMODB_POSTS_TABLE | Posts テーブル名 |
| AWS_REGION | DynamoDB リージョン |
| JWT_SECRET | JWT 署名シークレット |
| JWT_EXPIRES_IN | JWT 有効期限（例: `7d`）|

---

## 設計の詳細

各成果物の詳細はそれぞれのファイルを参照：

- コンポーネント定義・責務 → [components.md](components.md)
- メソッドシグネチャ・型定義 → [component-methods.md](component-methods.md)
- サービス定義・オーケストレーション → [services.md](services.md)
- 依存関係・データフロー図 → [component-dependency.md](component-dependency.md)
- ユニット・オブ・ワーク → [unit-of-work.md](unit-of-work.md)

---

## 旧版からの主な変更点（意味的再記述）

| 項目 | 旧 | 新（動的IP × AI 観点）|
|---|---|---|
| AICommentService | 称賛コメント生成 | **AINamakemonoService（動的IPの対話エンジン）**：人格 + RAG + 個別化記憶 |
| AICommentBubble | コメントの吹き出し | **NamakemonoBubble**（引用元を明記）|
| Posts.stamps フィールド | 存在 | **削除**（スタンプ機能は廃止）|
| Posts.aiCitationSource フィールド | 存在せず | **追加**（出典明記・ハルシネーション対策）|
| RAG引用ライブラリ | なし | **新規追加**（`lib/rag/`）|
| 個別化記憶 | なし | **新規追加**（`lib/memory/`）|
| サンドイッチUI（BrandFrame）| なし | **新規追加**（パンチライン保証）|

**構造変更**：なし。**3ユニットの境界も維持**。
