# アプリケーション設計 — Sloth Feed PoC（IP事業 × 動的IP × AI技術）

> ## 📌 CONSTRUCTION フェーズ着手時の必読事項（ピン留め）
> 実装計画策定の前に**必ず**以下 2 文書を参照すること：
> - **[security-review.md](security-review.md)** — OWASP Top 10 観点・Code Review チェックリスト 11 項目
> - **[version-management-review.md](version-management-review.md)** — 2024〜2025 インシデント対応・実装着手前に決定すべき 3 点（**Next.js 14.2.25+ / 15.2.3+ 必須**・**AWS は IAM ロール一本化**・**CI で `npm ci` + `npm audit`**）+ ベースライン 13 項目
>
> 両文書のチェックリストを実装タスクへ機械的に展開すること。

---

> **本ドキュメントの前提（2026-05-09 更新）**
> Issue #5 帰着により、Sloth Feed は IP事業として位置づけ直された。
> **技術選定（Next.js / Amazon Bedrock 経由の Claude / DynamoDB / JWT）は維持**しつつ、各ユニット・コンポーネントの**責務と意味を「動的IP × AI技術」文脈で再記述**している。構造変更はない。
> **注**：3回目サイクル（2026-05-09）で AI 呼び出しを Anthropic 直 API から **Amazon Bedrock 経由** に統一。

---

## 設計サマリー

| 項目 | 決定内容 |
|------|---------|
| フレームワーク | Next.js 14+ App Router / TypeScript |
| サービスレイヤー | `lib/services/` に分離（薄いコントローラ + サービスクラス）|
| DBアクセス | リポジトリパターン（`lib/repositories/`）|
| DynamoDB | **Posts のみ**（Users は Cognito 一本化により PoC 外、Phase 2 で補助テーブル検討）|
| AI サービス | **AIFilteringService**（仕事系投稿の弾き）と **AINamakemonoService**（旧 AICommentService、動的IPの対話）を分離 |
| 引用ソース戦略 | **PoC**：LLM の学習済み知識を信用（外部ナレッジベースなし）<br>**Phase 2**：S3 + Agentic Search による引用検証（FR-007 参照） |
| 個別化記憶 | **新規**：ユーザー履歴を AI に渡すコンテキスト構築 |
| **認証** | **Auth.js (NextAuth v5) + AWS Cognito User Pool**。`auth.ts` が設定の中心、`middleware.ts` は Auth.js の `auth` ヘルパに委譲 |
| **セッション保存** | **HttpOnly Cookie**（Auth.js デフォルト、XSS 耐性 ◎）|
| UIコンポーネント | `app/`（ページ）+ `components/`（再利用 UI、**サンドイッチUI構造**）|

---

## ディレクトリ構造

```
sloth-feed/
├── app/
│   ├── layout.tsx                  # Next.js App Router の root layout（必須）
│   ├── globals.css                 # 全画面共通スタイル
│   ├── providers.tsx               # 新規：SessionProvider（Auth.js）等を集約
│   ├── (main)/
│   │   ├── page.tsx                   # タイムライン（サンドイッチUI）
│   │   ├── post/
│   │   │   └── page.tsx               # 「仕事じゃないけど」投稿フォーム
│   │   └── my-posts/
│   │       └── page.tsx               # 自分の「仕事じゃないけど」投稿一覧
│   ├── auth/
│   │   └── (PoC 実装時に決定：Cognito Hosted UI / 自前ページ)
│   └── api/
│       ├── auth/
│       │   └── [...nextauth]/
│       │       └── route.ts            # Auth.js のルートハンドラ（auth.ts から再エクスポート）
│       ├── posts/
│       │   └── route.ts               # POST: 「仕事じゃないけど」投稿作成
│       ├── feed/
│       │   └── route.ts               # GET: タイムライン
│       └── my-posts/
│           └── route.ts               # GET: 自分の「仕事じゃないけど」投稿一覧
├── components/
│   ├── PostCard.tsx                   # サンドイッチUIで投稿+AIコメントを表示
│   ├── PostForm.tsx                   # 「仕事じゃないけど」投稿フォーム
│   ├── FeedList.tsx                   # PostCard のリスト
│   ├── NamakemonoBubble.tsx           # AIナマケモノコメントの吹き出し（旧 AICommentBubble）
│   ├── BrandFrame.tsx                 # 「仕事じゃないけど…世の中を変える」サンドイッチUIの上下フレーム
│   ├── FilteringFeedback.tsx          # 仕事系投稿除外時のフィードバック
│   ├── AuthForm.tsx                   # PoC 実装時に決定（Hosted UI 利用なら不要）
│   └── LoadingSpinner.tsx
├── lib/
│   ├── services/
│   │   ├── (auth.service.ts は廃止：Auth.js + Cognito 移行)
│   │   ├── post.service.ts
│   │   ├── feed.service.ts
│   │   ├── ai-filtering.service.ts    # 仕事系投稿の弾き
│   │   └── ai-namakemono.service.ts   # 旧 ai-comment.service.ts。動的IPの対話エンジン
│   ├── repositories/
│   │   ├── (user.repository.ts は PoC 外：Cognito 一本化)
│   │   └── post.repository.ts
│   ├── memory/                        # 新規：個別化記憶
│   │   └── user-history.ts            # ユーザー履歴の取得・コンテキスト構築
│   └── types/
│       └── index.ts
├── auth.ts                            # 新規：Auth.js (NextAuth v5) 設定
└── middleware.ts                      # Auth.js の auth を default export
```

---

## コンポーネント一覧

### バックエンド・サービス

| コンポーネント | パス | 主な責務（IP × 動的IP × AI 観点で再記述）|
|--------------|------|---------|
| AuthService | `lib/services/auth.service.ts` | 登録・ログイン・JWT発行。**IP のファン識別基盤** |
| PostService | `lib/services/post.service.ts` | ダメ投稿フローのオーケストレーション（フィルタリング → ナマケモノ対話 → 保存）|
| FeedService | `lib/services/feed.service.ts` | タイムライン・自分のダメ投稿一覧取得。**ファン共同体タイムライン** |
| **AIFilteringService** | `lib/services/ai-filtering.service.ts` | Bedrock Claude で仕事系投稿を弾く。**IPコンセプトの境界を守る** |
| **AINamakemonoService**（旧 AICommentService）| `lib/services/ai-namakemono.service.ts` | **動的IP の核**。Bedrock Claude + 個別化記憶 + LLM 学習済み知識からの引用で、ナマケモノとして個別化された肯定コメントを生成（Phase 2 で S3 + Agentic Search 拡張）|
| UserRepository | `lib/repositories/user.repository.ts` | Users テーブル CRUD |
| PostRepository | `lib/repositories/post.repository.ts` | Posts テーブル CRUD + GSI 検索 |
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
| LoadingSpinner | Bedrock Claude 呼び出し中のローディング |

---

## コアフロー：ダメ投稿作成（動的IP対話）

```
クライアント → middleware.ts（JWT検証）
             → POST /api/posts
             → PostService.createPost
                  → AIFilteringService.filterPost（Bedrock Claude）
                       ├─ 除外: 422 + reason を返す
                       └─ 通過:
                  → AINamakemonoService.generateResponse
                       ├─ UserHistory.getRecent（個別化記憶）
                       ├─ プロンプト構築（5経路言語化テンプレ + 想定引用源ヒント）
                       └─ Bedrock 経由で Claude モデル呼び出し（ナマケモノ人格、LLM 学習済み知識から引用生成）
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
| AWS_REGION | AWS リージョン（Bedrock・DynamoDB・Cognito 共通）|
| BEDROCK_MODEL_ID | 利用する Claude モデル ID（例：`anthropic.claude-3-5-sonnet-20241022-v2:0` または最新版）|
| AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY または IAM ロール | 開発時はキー、本番ではタスクロール／インスタンスロール推奨 |
| ~~DYNAMODB_USERS_TABLE~~ | **3回目サイクルで PoC 外**（Cognito 一本化）|
| DYNAMODB_POSTS_TABLE | Posts テーブル名 |
| COGNITO_USER_POOL_ID | Cognito ユーザープール ID |
| COGNITO_APP_CLIENT_ID | Cognito App Client ID |
| COGNITO_APP_CLIENT_SECRET | Cognito App Client Secret（Auth.js OAuth が要求）|
| AUTH_SECRET | Auth.js Cookie 暗号化用ランダム文字列 |
| NEXTAUTH_URL | アプリ Base URL（例: `http://localhost:3000`）|

---

## 設計の詳細

各成果物の詳細はそれぞれのファイルを参照：

- コンポーネント定義・責務 → [components.md](components.md)
- メソッドシグネチャ・型定義 → [component-methods.md](component-methods.md)
- サービス定義・オーケストレーション → [services.md](services.md)
- 依存関係・データフロー図 → [component-dependency.md](component-dependency.md)
- ユニット・オブ・ワーク → [unit-of-work.md](unit-of-work.md)
- **技術選択のセキュリティレビュー → [security-review.md](security-review.md)**
- **バージョン管理レビュー（2024〜2025 インシデント対応）→ [version-management-review.md](version-management-review.md)**

---

## 旧版からの主な変更点（意味的再記述）

| 項目 | 旧 | 新（動的IP × AI 観点）|
|---|---|---|
| AICommentService | 称賛コメント生成 | **AINamakemonoService（動的IPの対話エンジン）**：人格 + 個別化記憶 + LLM 学習済み引用（Phase 2 で S3 Agentic Search 拡張）|
| AICommentBubble | コメントの吹き出し | **NamakemonoBubble**（引用元を明記）|
| Posts.stamps フィールド | 存在 | **削除**（スタンプ機能は廃止）|
| Posts.aiCitationSource フィールド | 存在せず | **追加**（出典明記・ハルシネーション対策）|
| 引用ソース | なし | **PoC は LLM の学習済み知識を信用**（外部 KB なし）<br>**Phase 2 で S3 + Agentic Search**（FR-007）|
| 個別化記憶 | なし | **新規追加**（`lib/memory/`）|
| サンドイッチUI（BrandFrame）| なし | **新規追加**（パンチライン保証）|
| **認証** | 自前 AuthService + bcrypt + JWT | **Auth.js + Cognito User Pool（OAuth/OIDC）**（3回目サイクルで切替）|
| **セッション** | localStorage + Bearer Token | **HttpOnly Cookie**（Auth.js）|
| **Users テーブル** | DynamoDB Users | **PoC では作成しない**（Cognito 一本化、Phase 2 で補助テーブル検討）|

**構造変更**：なし。**3ユニットの境界も維持**。
